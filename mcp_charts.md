# MCP Charts & Visualisation Tool

**Project:** outlook-ai-agent-mcp
**Tool name:** `generate_chart`
**Output:** PNG image files (bar, line, pie)
**Trigger:** User asks for charts from email data, task lists, or calendar

-----

## PART A — How It Works

```
User: "Show me a pie chart of my email senders this week"
        │
        ▼
LibreChat LLM → calls list_emails tool (gets raw data)
        │
        ▼
LibreChat LLM → calls generate_chart tool
        (passes data + chart_type + title)
        │
        ▼
generate_chart tool:
  - builds chart using matplotlib
  - saves as PNG to temp_attachments/charts/
  - returns file path + base64 encoded image
        │
        ▼
LibreChat displays the chart inline in chat
```

-----

## PART B — Prerequisites

### Install library

```bash
pip install matplotlib
```

### Add to `requirements.txt`

```txt
matplotlib>=3.9.0
```

### Add to `.env`

```env
# =================================================================
# CHARTS
# =================================================================
CHARTS_OUTPUT_DIR=./temp_attachments/charts
CHARTS_MAX_ITEMS=20
```

### Add to `config/settings.py` inside `__init__`:

```python
        # ----- Charts -----
        self.CHARTS_OUTPUT_DIR: str = _optional_env(
            "CHARTS_OUTPUT_DIR", "./temp_attachments/charts"
        )
        self.CHARTS_MAX_ITEMS: int = int(
            _optional_env("CHARTS_MAX_ITEMS", "20")
        )
```

### Create charts output folder

```bash
mkdir -p /opt/FiGPT_OutlookMCP/temp_attachments/charts
```

Add to `.gitignore`:

```gitignore
temp_attachments/charts/*
!temp_attachments/charts/.gitkeep
```

Create placeholder:

```bash
touch /opt/FiGPT_OutlookMCP/temp_attachments/charts/.gitkeep
```

-----

## PART C — New File: `tools/chart_tools.py`

```python
"""
chart_tools.py
==============
MCP tool for generating charts and visualisations from email data,
task lists, calendar events, or any structured data the user provides.

Why this file exists:
    LibreChat's LLM can produce SVG code for charts, but cannot
    generate actual downloadable image files. This tool fills that
    gap — it takes structured data, builds a real chart using
    matplotlib, saves it as a PNG file, and returns both the file
    path and a base64-encoded version for inline display.

Supported chart types:
    - bar   : Compare values across categories
              e.g. email count per sender, tasks per owner
    - line  : Show trends over time
              e.g. emails received per day over a week
    - pie   : Show proportions/distribution
              e.g. email split by folder, tasks by status

Design notes:
    - No external API calls — matplotlib runs entirely locally.
    - Charts saved to temp_attachments/charts/ as PNG files.
    - Base64 encoding allows LibreChat to display inline.
    - matplotlib uses 'Agg' backend (no display/GUI required)
      which is correct for server-side rendering on Linux.
    - File names include timestamp to avoid collisions.
"""

# ---------------------------------------------------------------------------
# Imports
# ---------------------------------------------------------------------------
import os
import uuid
import base64
from pathlib import Path
from datetime import datetime

# Use non-interactive backend BEFORE importing pyplot
# This is critical on a Linux server with no display (FGSV297)
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches

from tools.mcp_instance import mcp
from config.settings import settings
from utils.error_handler import format_tool_error
from utils.logger import get_logger

logger = get_logger(__name__)

# Fichtner brand-aligned colour palette
# Professional blues, greys and accent colours
CHART_COLOURS = [
    "#0078D4",  # Microsoft Blue — primary
    "#50E6FF",  # Azure Cyan
    "#FFB900",  # Amber
    "#E74856",  # Red
    "#00CC6A",  # Green
    "#8764B8",  # Purple
    "#FF8C00",  # Orange
    "#004E8C",  # Dark Blue
    "#038387",  # Teal
    "#C239B3",  # Magenta
]


# ---------------------------------------------------------------------------
# Helper: ensure output directory exists
# ---------------------------------------------------------------------------
def _ensure_chart_dir() -> Path:
    """
    Create the charts output directory if it doesn't exist.

    Returns:
        Path: The absolute path to the charts output directory.
    """
    chart_dir = Path(settings.CHARTS_OUTPUT_DIR).resolve()
    chart_dir.mkdir(parents=True, exist_ok=True)
    return chart_dir


# ---------------------------------------------------------------------------
# Helper: save figure and encode to base64
# ---------------------------------------------------------------------------
def _save_and_encode(fig: plt.Figure, filename: str) -> tuple[str, str]:
    """
    Save a matplotlib figure as PNG and encode it to base64.

    Args:
        fig (plt.Figure): The matplotlib figure to save.
        filename (str): Output file name (without path).

    Returns:
        tuple[str, str]: (file_path, base64_string)
            - file_path: absolute path to the saved PNG
            - base64_string: base64 encoded PNG for inline display
    """
    chart_dir = _ensure_chart_dir()
    file_path = chart_dir / filename

    # Save with high DPI for crisp output
    fig.savefig(
        file_path,
        format="png",
        dpi=150,
        bbox_inches="tight",
        facecolor="white",
        edgecolor="none",
    )
    plt.close(fig)

    # Encode to base64 for inline display in LibreChat
    with open(file_path, "rb") as f:
        encoded = base64.b64encode(f.read()).decode("utf-8")

    logger.info(f"Chart saved: {file_path}")
    return str(file_path), encoded


# ---------------------------------------------------------------------------
# Helper: generate unique filename
# ---------------------------------------------------------------------------
def _make_filename(chart_type: str) -> str:
    """
    Generate a unique timestamped filename for a chart PNG.

    Args:
        chart_type (str): Chart type label used in the filename.

    Returns:
        str: e.g. "bar_chart_20260715_143022_a3f1.png"
    """
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    short_id = str(uuid.uuid4())[:4]
    return f"{chart_type}_chart_{timestamp}_{short_id}.png"


# ---------------------------------------------------------------------------
# Tool: generate_chart
# ---------------------------------------------------------------------------
@mcp.tool
async def generate_chart(
    chart_type: str,
    labels: str,
    values: str,
    title: str = "",
    x_label: str = "",
    y_label: str = "",
) -> dict:
    """
    Generate a chart (bar, line, or pie) from provided data and
    save it as a downloadable PNG image file.

    Use this tool when the user asks things like:
    - "Show me a bar chart of emails by sender"
    - "Generate a pie chart of my task status distribution"
    - "Plot a line chart of emails received per day this week"
    - "Visualise my calendar events by day as a bar chart"
    - "Create a chart from this data"

    This tool is typically called AFTER another tool provides data
    (e.g. list_emails, list_tasks, list_calendar_events). The LLM
    aggregates the data from those results and passes it here.

    Args:
        chart_type (str): Type of chart to generate:
                           - "bar"  : vertical bar chart
                           - "line" : line/trend chart
                           - "pie"  : pie/donut chart
        labels (str): Comma-separated category labels.
                       e.g. "John Smith, Jane Doe, HR Team"
                       e.g. "Mon, Tue, Wed, Thu, Fri"
        values (str): Comma-separated numeric values matching labels.
                       e.g. "15, 8, 3"
                       e.g. "4, 7, 2, 9, 5"
        title (str): Chart title displayed at the top.
                      e.g. "Emails by Sender — Last 7 Days"
        x_label (str): X-axis label (for bar and line charts).
                        e.g. "Sender", "Date"
        y_label (str): Y-axis label (for bar and line charts).
                        e.g. "Email Count", "Number of Tasks"

    Returns:
        dict with file path, base64 image, and display instruction.
    """
    logger.info(
        f"Tool called: generate_chart "
        f"(type={chart_type}, title='{title}')"
    )

    try:
        # Step 1: Parse and validate inputs
        chart_type_clean = chart_type.strip().lower()

        if chart_type_clean not in ("bar", "line", "pie"):
            return {
                "error": True,
                "message": (
                    f"Unsupported chart type: '{chart_type}'. "
                    f"Please use 'bar', 'line', or 'pie'."
                ),
            }

        # Parse labels and values from comma-separated strings
        label_list = [l.strip() for l in labels.split(",") if l.strip()]
        value_list_raw = [v.strip() for v in values.split(",") if v.strip()]

        if not label_list or not value_list_raw:
            return {
                "error": True,
                "message": "Labels and values cannot be empty.",
            }

        if len(label_list) != len(value_list_raw):
            return {
                "error": True,
                "message": (
                    f"Mismatch: {len(label_list)} labels but "
                    f"{len(value_list_raw)} values. They must match."
                ),
            }

        # Convert values to float — fail clearly if non-numeric
        try:
            value_list = [float(v) for v in value_list_raw]
        except ValueError as ve:
            return {
                "error": True,
                "message": (
                    f"Values must be numbers. Got: {value_list_raw}. "
                    f"Error: {ve}"
                ),
            }

        # Cap items to avoid unreadable charts
        max_items = settings.CHARTS_MAX_ITEMS
        if len(label_list) > max_items:
            label_list = label_list[:max_items]
            value_list = value_list[:max_items]
            logger.info(f"Truncated chart data to {max_items} items")

        # Step 2: Apply consistent chart styling
        plt.rcParams.update({
            "font.family": "DejaVu Sans",
            "font.size": 11,
            "axes.titlesize": 14,
            "axes.titleweight": "bold",
            "axes.titlepad": 16,
            "axes.spines.top": False,
            "axes.spines.right": False,
            "figure.facecolor": "white",
            "axes.facecolor": "#f8f9fa",
            "axes.grid": True,
            "grid.alpha": 0.4,
            "grid.linestyle": "--",
        })

        # Step 3: Generate the correct chart type
        if chart_type_clean == "bar":
            file_path, encoded = _generate_bar_chart(
                label_list, value_list, title, x_label, y_label
            )

        elif chart_type_clean == "line":
            file_path, encoded = _generate_line_chart(
                label_list, value_list, title, x_label, y_label
            )

        elif chart_type_clean == "pie":
            file_path, encoded = _generate_pie_chart(
                label_list, value_list, title
            )

        # Step 4: Return result with base64 for inline display
        file_name = Path(file_path).name

        return {
            "chart_type": chart_type_clean,
            "title": title,
            "file_name": file_name,
            "file_path": file_path,
            "data_points": len(label_list),
            "image_base64": encoded,
            "instruction": (
                f"A {chart_type_clean} chart titled '{title}' has been "
                f"generated with {len(label_list)} data points.\n\n"
                f"Display the chart to the user using this markdown:\n\n"
                f"**📊 {title}**\n\n"
                f"![{title}](data:image/png;base64,{encoded})\n\n"
                f"*Chart saved as: `{file_name}`*\n\n"
                f"Then summarise the key insight from the chart data in "
                f"1-2 sentences."
            ),
        }

    except Exception as exc:
        return format_tool_error(exc, tool_name="generate_chart", logger=logger)


# ---------------------------------------------------------------------------
# Internal: Bar chart generator
# ---------------------------------------------------------------------------
def _generate_bar_chart(
    labels: list,
    values: list,
    title: str,
    x_label: str,
    y_label: str,
) -> tuple[str, str]:
    """
    Generate a professional vertical bar chart.

    Args:
        labels (list): Category labels for x-axis.
        values (list): Numeric values for each bar.
        title (str): Chart title.
        x_label (str): X-axis label.
        y_label (str): Y-axis label.

    Returns:
        tuple[str, str]: (file_path, base64_encoded_image)
    """
    # Dynamic figure width based on number of bars
    fig_width = max(8, len(labels) * 0.9)
    fig, ax = plt.subplots(figsize=(fig_width, 5))

    # Use colour palette — cycle if more bars than colours
    colours = [
        CHART_COLOURS[i % len(CHART_COLOURS)]
        for i in range(len(labels))
    ]

    bars = ax.bar(
        labels,
        values,
        color=colours,
        edgecolor="white",
        linewidth=1.5,
        width=0.6,
    )

    # Add value labels on top of each bar
    for bar, value in zip(bars, values):
        ax.text(
            bar.get_x() + bar.get_width() / 2,
            bar.get_height() + max(values) * 0.01,
            f"{value:,.0f}" if value == int(value) else f"{value:,.1f}",
            ha="center",
            va="bottom",
            fontsize=10,
            fontweight="bold",
            color="#333333",
        )

    ax.set_title(title or "Bar Chart", pad=16)
    if x_label:
        ax.set_xlabel(x_label, labelpad=8)
    if y_label:
        ax.set_ylabel(y_label, labelpad=8)

    # Rotate x labels if many items
    if len(labels) > 6:
        plt.xticks(rotation=30, ha="right")

    ax.set_ylim(0, max(values) * 1.15)
    plt.tight_layout()

    filename = _make_filename("bar")
    return _save_and_encode(fig, filename)


# ---------------------------------------------------------------------------
# Internal: Line chart generator
# ---------------------------------------------------------------------------
def _generate_line_chart(
    labels: list,
    values: list,
    title: str,
    x_label: str,
    y_label: str,
) -> tuple[str, str]:
    """
    Generate a professional line/trend chart.

    Args:
        labels (list): Category labels for x-axis (time points).
        values (list): Numeric values at each time point.
        title (str): Chart title.
        x_label (str): X-axis label.
        y_label (str): Y-axis label.

    Returns:
        tuple[str, str]: (file_path, base64_encoded_image)
    """
    fig, ax = plt.subplots(figsize=(10, 5))

    x_positions = range(len(labels))

    # Main line
    ax.plot(
        x_positions,
        values,
        color=CHART_COLOURS[0],
        linewidth=2.5,
        marker="o",
        markersize=7,
        markerfacecolor="white",
        markeredgecolor=CHART_COLOURS[0],
        markeredgewidth=2.5,
        zorder=3,
    )

    # Shaded area under the line
    ax.fill_between(
        x_positions,
        values,
        alpha=0.12,
        color=CHART_COLOURS[0],
    )

    # Value labels at each data point
    for i, (x, y) in enumerate(zip(x_positions, values)):
        ax.annotate(
            f"{y:,.0f}" if y == int(y) else f"{y:,.1f}",
            (x, y),
            textcoords="offset points",
            xytext=(0, 10),
            ha="center",
            fontsize=9,
            color="#333333",
        )

    ax.set_xticks(x_positions)
    ax.set_xticklabels(labels, rotation=30 if len(labels) > 6 else 0, ha="right")
    ax.set_title(title or "Line Chart", pad=16)
    if x_label:
        ax.set_xlabel(x_label, labelpad=8)
    if y_label:
        ax.set_ylabel(y_label, labelpad=8)

    ax.set_ylim(0, max(values) * 1.2)
    plt.tight_layout()

    filename = _make_filename("line")
    return _save_and_encode(fig, filename)


# ---------------------------------------------------------------------------
# Internal: Pie chart generator
# ---------------------------------------------------------------------------
def _generate_pie_chart(
    labels: list,
    values: list,
    title: str,
) -> tuple[str, str]:
    """
    Generate a professional pie chart with percentage labels.

    Args:
        labels (list): Segment labels.
        values (list): Numeric values for each segment.
        title (str): Chart title.

    Returns:
        tuple[str, str]: (file_path, base64_encoded_image)
    """
    fig, ax = plt.subplots(figsize=(9, 7))

    colours = [
        CHART_COLOURS[i % len(CHART_COLOURS)]
        for i in range(len(labels))
    ]

    # Slightly explode the largest slice for emphasis
    max_idx = values.index(max(values))
    explode = [0.05 if i == max_idx else 0 for i in range(len(values))]

    wedges, texts, autotexts = ax.pie(
        values,
        labels=None,         # Labels via legend instead of on-chart
        autopct="%1.1f%%",
        colors=colours,
        explode=explode,
        startangle=90,
        pctdistance=0.75,
        wedgeprops={
            "linewidth": 2,
            "edgecolor": "white",
        },
    )

    # Style percentage text
    for autotext in autotexts:
        autotext.set_fontsize(10)
        autotext.set_fontweight("bold")
        autotext.set_color("white")

    # Build legend with value + percentage
    total = sum(values)
    legend_labels = [
        f"{label}  ({val:,.0f} — {val/total*100:.1f}%)"
        for label, val in zip(labels, values)
    ]
    legend_patches = [
        mpatches.Patch(color=colours[i], label=legend_labels[i])
        for i in range(len(labels))
    ]

    ax.legend(
        handles=legend_patches,
        loc="lower center",
        bbox_to_anchor=(0.5, -0.15),
        ncol=min(3, len(labels)),
        frameon=False,
        fontsize=9,
    )

    ax.set_title(title or "Pie Chart", pad=20)
    plt.tight_layout()

    filename = _make_filename("pie")
    return _save_and_encode(fig, filename)
```

-----

## PART D — Register in `server.py`

Add one line to the Phase 3 import block:

```python
        # Chart tools — generate_chart
        from tools import chart_tools  # noqa: F401
```

-----

## PART E — Test File: `tests/test_charts.py`

```python
"""
test_charts.py
==============
Tests for tools/chart_tools.py — generate_chart tool.

How to run:
    python -m pytest tests/test_charts.py -v
"""

import pytest
from unittest.mock import patch, MagicMock
from pathlib import Path


class TestGenerateChart:
    """Tests for generate_chart tool."""

    @pytest.mark.asyncio
    async def test_bar_chart_generates_successfully(self):
        """
        generate_chart() with type='bar' should return a file path,
        base64 string, and instruction containing markdown image tag.
        """
        with patch("tools.chart_tools._generate_bar_chart") as mock_bar:
            mock_bar.return_value = ("/tmp/bar_chart_test.png", "base64string==")

            from tools.chart_tools import generate_chart
            result = await generate_chart(
                chart_type="bar",
                labels="John, Jane, HR",
                values="15, 8, 3",
                title="Emails by Sender",
                y_label="Email Count",
            )

        assert result.get("error") is None or "error" not in result
        assert result["chart_type"] == "bar"
        assert result["data_points"] == 3
        assert "base64string" in result["image_base64"]
        assert "data:image/png;base64," in result["instruction"]

    @pytest.mark.asyncio
    async def test_pie_chart_generates_successfully(self):
        """
        generate_chart() with type='pie' should produce a pie chart.
        """
        with patch("tools.chart_tools._generate_pie_chart") as mock_pie:
            mock_pie.return_value = ("/tmp/pie_chart_test.png", "piebase64==")

            from tools.chart_tools import generate_chart
            result = await generate_chart(
                chart_type="pie",
                labels="Inbox, Drafts, Sent",
                values="120, 5, 45",
                title="Email Distribution by Folder",
            )

        assert result["chart_type"] == "pie"
        assert result["data_points"] == 3

    @pytest.mark.asyncio
    async def test_line_chart_generates_successfully(self):
        """
        generate_chart() with type='line' should produce a line chart.
        """
        with patch("tools.chart_tools._generate_line_chart") as mock_line:
            mock_line.return_value = ("/tmp/line_chart_test.png", "linebase64==")

            from tools.chart_tools import generate_chart
            result = await generate_chart(
                chart_type="line",
                labels="Mon, Tue, Wed, Thu, Fri",
                values="4, 7, 2, 9, 5",
                title="Emails Received Per Day",
                x_label="Day",
                y_label="Count",
            )

        assert result["chart_type"] == "line"
        assert result["data_points"] == 5

    @pytest.mark.asyncio
    async def test_returns_error_for_invalid_chart_type(self):
        """
        generate_chart() should return an error dict for
        unsupported chart types.
        """
        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="scatter",
            labels="A, B",
            values="1, 2",
            title="Test",
        )

        assert result.get("error") is True
        assert "scatter" in result["message"]

    @pytest.mark.asyncio
    async def test_returns_error_for_mismatched_labels_values(self):
        """
        generate_chart() should return an error when label count
        does not match value count.
        """
        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="bar",
            labels="A, B, C",
            values="1, 2",
            title="Mismatch Test",
        )

        assert result.get("error") is True
        assert "Mismatch" in result["message"]

    @pytest.mark.asyncio
    async def test_returns_error_for_non_numeric_values(self):
        """
        generate_chart() should return a clear error when
        values contain non-numeric strings.
        """
        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="bar",
            labels="A, B",
            values="10, twenty",
            title="Bad Values Test",
        )

        assert result.get("error") is True
        assert "numbers" in result["message"].lower()

    @pytest.mark.asyncio
    async def test_returns_error_for_empty_labels(self):
        """
        generate_chart() should return an error when labels
        or values are empty.
        """
        from tools.chart_tools import generate_chart
        result = await generate_chart(
            chart_type="pie",
            labels="",
            values="",
            title="Empty Test",
        )

        assert result.get("error") is True
```

-----

## PART F — Example Prompts in LibreChat

After implementation, these prompts work end to end:

```
List my last 20 emails and generate a bar chart showing
the top 5 senders by email count

Show me a pie chart of my calendar events this week grouped by type

Generate a line chart of emails I received each day this week

Create a bar chart comparing my task count in To-Do vs Planner
```

-----

## PART G — Summary

|Item              |Detail                                                           |
|------------------|-----------------------------------------------------------------|
|New file          |`tools/chart_tools.py`                                           |
|New test file     |`tests/test_charts.py`                                           |
|Library           |`matplotlib` — local, no API calls                               |
|Output format     |PNG — 150 DPI                                                    |
|Inline display    |base64 embedded in markdown `![title](data:image/png;base64,...)`|
|Supported types   |bar, line, pie                                                   |
|Colours           |Professional blue palette — Fichtner aligned                     |
|Backend           |`Agg` — required for headless Linux server                       |
|Max data points   |20 (configurable via `.env`)                                     |
|Storage           |`temp_attachments/charts/` — not committed to Azure DevOps       |
|Tools after adding|34 total                                                         |

-----

*Apply changes, restart server, verify health check shows 34 tools.*

---

You’re right — LibreChat doesn’t render data:image/png;base64,... inline. It treats it as plain text. Here are your real options:

Options
Option 1 — SVG (Best fit for LibreChat) ✅ Recommended
Generate charts as SVG using matplotlib — LibreChat renders SVG natively in chat. No base64, no file path shown, just the chart.
Option 2 — HTML file
Save chart as a self-contained .html file with the chart embedded. User clicks a link to open it in browser.
Option 3 — Plotly HTML
Use plotly instead of matplotlib — generates interactive HTML charts. User opens in browser. More features (zoom, hover tooltips).
Option 4 — ASCII chart in chat
Render a simple text-based bar chart directly in markdown. No files, no images — works everywhere. Limited visual quality.

Recommendation
Go with Option 1 (SVG) for inline viewing + Option 2 (HTML) as downloadable backup.
SVG renders natively in LibreChat’s markdown. Clean, no lengthy addresses, no base64 strings shown.

Fix — Update tools/chart_tools.py
Step 1 — Change output format to SVG
Replace _save_and_encode function:

def _save_as_svg(fig: plt.Figure, filename: str) -> tuple[str, str]:
    """
    Save a matplotlib figure as SVG and read it as a string.
    SVG renders natively in LibreChat's markdown.

    Args:
        fig (plt.Figure): The matplotlib figure to save.
        filename (str): Output file name (without path).

    Returns:
        tuple[str, str]: (file_path, svg_string)
    """
    chart_dir = _ensure_chart_dir()
    file_path = chart_dir / filename

    fig.savefig(
        file_path,
        format="svg",
        bbox_inches="tight",
        facecolor="white",
    )
    plt.close(fig)

    # Read SVG as string for inline rendering
    svg_string = file_path.read_text(encoding="utf-8")

    logger.info(f"SVG chart saved: {file_path}")
    return str(file_path), svg_string


Step 2 — Change filename generator

def _make_filename(chart_type: str) -> str:
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    short_id = str(uuid.uuid4())[:4]
    return f"{chart_type}_chart_{timestamp}_{short_id}.svg"  # .svg not .png


Step 3 — Update all three internal generators
In _generate_bar_chart, _generate_line_chart, _generate_pie_chart — change the last two lines of each from:

    filename = _make_filename("bar")
    return _save_and_encode(fig, filename)


To:

    filename = _make_filename("bar")
    return _save_as_svg(fig, filename)


Same for line and pie.
Step 4 — Update generate_chart tool return
Find the instruction field in the return block and replace:

            "instruction": (
                f"A {chart_type_clean} chart titled '{title}' has been "
                f"generated with {len(label_list)} data points.\n\n"
                f"Display the chart to the user using this markdown:\n\n"
                f"**📊 {title}**\n\n"
                f"![{title}](data:image/png;base64,{encoded})\n\n"
                f"*Chart saved as: `{file_name}`*\n\n"
                f"Then summarise the key insight from the chart data in "
                f"1-2 sentences."
            ),


Replace with:

            "instruction": (
                f"Display this SVG chart directly in the chat — "
                f"render it as raw SVG markup, do not wrap it in "
                f"a code block:\n\n"
                f"**📊 {title}**\n\n"
                f"{svg_content}\n\n"
                f"Then in 1-2 sentences summarise the key insight "
                f"from this data."
            ),


Also rename encoded to svg_content in the variable assignment:

        file_path, svg_content = _generate_bar_chart(...)
        # or line / pie


And update the return dict:

        return {
            "chart_type": chart_type_clean,
            "title": title,
            "file_name": file_name,
            "file_path": file_path,
            "data_points": len(label_list),
            "instruction": (...)  # updated above
        }


Remove "image_base64" from the return dict entirely.

Result after fix
LibreChat will render the SVG directly in chat — the chart appears as a proper visual, no long addresses, no base64 strings. The SVG file is also saved to temp_attachments/charts/ as a backup.
Restart server and test:

List my last 10 emails and show a bar chart of top 5 senders







