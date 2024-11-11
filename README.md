
# QuackReports 

*QuackReports* is a robust, open-source toolkit for creating custom reporting engines. 🦆

## Main Features

Create custom report engines that handle report generation tasks. The capabilities of these engines extend to the following:

- 📃 **Static Engines:** Intuitively define a single-use, static report engine to render ad-hoc report requests.
- 🔄 **Dynamic Engines:** Support repeated, flexible report generation by creating dynamic report layouts and elements.
- 🔗 **Integrations:** Use familiar analytics libraries to define report components (e.g., Pandas, Matplotlib, and more).
- 🎨 **Formatting:** Control the look and feel of reports and their components globally and locally with ease.

## Installation

To install *QuackReports*, use pip:

```bash
pip install quackreports
```

## Quick Start

In this quick start guide, we'll create both a static and a dynamic sales report using the same dataset and business scenario. This will help you understand how to build upon a static report to make it dynamic with *QuackReports*.

### Business Scenario

Imagine you're an analyst at a retail company, and you need to generate sales reports. You have sales data for different regions over several months. Initially, you need a static report summarizing overall sales. Later, you want to generate customized reports for specific regions and months dynamically.

### Dataset

We'll use the following sample sales data:

```python
import pandas as pd

# Sample Sales Data
data = {
    "Region": ["North", "South", "East", "West"] * 3,
    "Month": ["January"] * 4 + ["February"] * 4 + ["March"] * 4,
    "Total Sales": [15000, 12000, 13000, 14000, 16000, 11000, 15000, 13000, 17000, 12500, 16000, 13500],
    "Units Sold": [300, 240, 260, 280, 320, 220, 300, 260, 340, 250, 320, 270],
    "Profit": [4500, 3600, 3900, 4200, 4800, 3300, 4500, 3900, 5100, 3750, 4800, 4050]
}
df = pd.DataFrame(data)
```

### Static Report Engine

First, we'll create a static sales report that includes a cover page, a sales data table, and a sales trend plot.

```python
import matplotlib.pyplot as plt
from quackreports import ReportEngine, Report
from quackreports.elements import Image, Text, Table, Plot

# Elements for the Cover Page
logo = Image("logo.png", dim=(100, 100))
title = Text("Company Sales Report", font_size=28, bold=True)
subtitle = Text("Quarterly Overview", font_size=22, italic=True)

# Sales Data Table
table = Table(df)

# Sales Trend Plot
fig, ax = plt.subplots()
monthly_sales = df.groupby("Month")["Total Sales"].sum()
ax.plot(monthly_sales.index, monthly_sales.values, marker='o', color='blue')
ax.set_title("Total Sales Over Months")
ax.set_xlabel("Month")
ax.set_ylabel("Total Sales ($)")
plot = Plot(fig)

# Report Layout
layout = [
    # Cover Page Section
    [
        logo,
        title,
        subtitle
    ],
    # Sales Data Section
    [
        Text("Overall Sales Data", font_size=24, bold=True),
        table
    ],
    # Sales Trend Section
    [
        Text("Sales Trend", font_size=24, bold=True),
        plot
    ]
]

# Create and Render Report
report = Report(layout)
report_engine = ReportEngine(report)
rendered_report = report_engine.run()
rendered_report.save("static_sales_report.pdf")
```

This static report provides an overview of the company's sales across all regions and months. It includes:

- A **cover page** with the company logo and report title.
- A **sales data table** showing detailed sales figures.
- A **sales trend plot** visualizing total sales over the months.

### Dynamic Report Engine

Now, we'll enhance the report by making it dynamic. We'll introduce configurations that allow us to generate customized reports based on user input, such as selecting specific regions and months.

#### Step 1: Define the Configuration Model

We use a Pydantic model to define the configuration schema and include validators to ensure valid input.

```python
from pydantic import BaseModel, Field, validator

class ReportConfig(BaseModel):
    regions: list = Field(default=[], description="List of regions to include in the report")
    months: list = Field(default=[], description="List of months to include in the report")
    include_profit: bool = Field(default=False, description="Include profit data in the report")

    @validator("regions", each_item=True)
    def validate_regions(cls, value):
        valid_regions = df["Region"].unique()
        if value not in valid_regions:
            raise ValueError(f"Region '{value}' does not exist.")
        return value

    @validator("months", each_item=True)
    def validate_months(cls, value):
        valid_months = df["Month"].unique()
        if value not in valid_months:
            raise ValueError(f"Month '{value}' is not valid.")
        return value

    @validator("regions", "months")
    def validate_not_empty(cls, value, field):
        if not value:
            raise ValueError(f"{field.name.capitalize()} list cannot be empty.")
        return value
```

#### Step 2: Create Dynamic Elements

We define functions that generate report elements based on the configuration.

```python
def dynamic_title(config):
    regions = ', '.join(config.regions)
    return Text(f"Sales Report for {regions} Region(s)", font_size=28, bold=True)

def dynamic_subtitle(config):
    months = ', '.join(config.months)
    return Text(f"Months: {months}", font_size=22, italic=True)

def dynamic_table(config):
    # Filter data based on configuration
    filtered_data = df[
        df["Region"].isin(config.regions) &
        df["Month"].isin(config.months)
    ].copy()
    if not config.include_profit:
        filtered_data = filtered_data.drop(columns=["Profit"])
    return Table(filtered_data.reset_index(drop=True))

def dynamic_plot(config):
    # Aggregate data for the plot
    filtered_data = df[
        df["Region"].isin(config.regions) &
        df["Month"].isin(config.months)
    ]
    sales_by_month = filtered_data.groupby("Month")["Total Sales"].sum().reindex(config.months)
    # Create plot
    fig, ax = plt.subplots()
    ax.plot(sales_by_month.index, sales_by_month.values, marker='o', color='green')
    ax.set_title("Total Sales Over Selected Months")
    ax.set_xlabel("Month")
    ax.set_ylabel("Total Sales ($)")
    return Plot(fig)

def profit_section(config):
    if config.include_profit:
        total_profit = df[
            df["Region"].isin(config.regions) &
            df["Month"].isin(config.months)
        ]["Profit"].sum()
        profit_text = Text(f"Total Profit: ${total_profit:,.2f}", font_size=18, bold=True)
        return [
            Text("Profit Summary", font_size=24, bold=True),
            profit_text
        ]
    return []
```

#### Step 3: Define the Dynamic Layout

We create a layout function that builds the report layout based on the configuration.

```python
def report_layout(config):
    layout = [
        # Cover Page Section
        [
            logo,
            dynamic_title,
            dynamic_subtitle
        ],
        # Sales Data Section
        [
            Text("Filtered Sales Data", font_size=24, bold=True),
            dynamic_table
        ],
        # Sales Trend Section
        [
            Text("Sales Trend", font_size=24, bold=True),
            dynamic_plot
        ]
    ]
    # Add Profit Section if requested
    profit = profit_section(config)
    if profit:
        layout.append(profit)
    return layout
```

#### Step 4: Generate Reports with Different Configurations

Now, we can generate customized reports by passing different configurations.

```python
# Create a report instance
report = Report(report_layout)

# Initialize the report engine
report_engine = ReportEngine(report, config_model=ReportConfig)

# Configuration 1: Report for North and East regions for January and February, including profit
config1 = {
    "regions": ["North", "East"],
    "months": ["January", "February"],
    "include_profit": True
}

# Generate and save the report
rendered_report1 = report_engine.run(config1)
rendered_report1.save("dynamic_report_north_east.pdf")

# Configuration 2: Report for all regions for March, excluding profit
config2 = {
    "regions": ["North", "South", "East", "West"],
    "months": ["March"],
    "include_profit": False
}

rendered_report2 = report_engine.run(config2)
rendered_report2.save("dynamic_report_march_all_regions.pdf")
```

#### Step 5: Handling Invalid Configurations

The validators in the `ReportConfig` model ensure that invalid configurations are caught before generating the report.

```python
# Invalid Configuration: Empty regions list
invalid_config = {
    "regions": [],
    "months": ["January", "February"],
    "include_profit": False
}

try:
    rendered_report_invalid = report_engine.run(invalid_config)
except ValueError as e:
    print(f"Validation Error: {e}")
```

Output:

```
Validation Error: Regions list cannot be empty.
```

### Explanation of Dynamic Enhancements

In the dynamic report, we've added the following features to make it dynamic:

- **Configurable Data Selection:** Users can select specific regions and months for the report.
- **Dynamic Elements:** The title, subtitle, table, and plot adjust based on the configuration.
- **Conditional Content:** The profit column is included in the table and a profit summary section is added only if `include_profit` is `True`.
- **Input Validation:** Validators ensure that the provided regions and months are valid and not empty.

### Summary

By building upon the static report, we've demonstrated how to use *QuackReports* to create flexible and customizable reports that adapt to different user requirements. The scenario carries through the entire README, showing how you can start with a basic report and enhance it with dynamic features.

## Documentation

For detailed documentation and advanced usage, please visit our [Documentation Page](link-to-documentation).

## Contributing

We welcome contributions! Please read our [Contributing Guidelines](CONTRIBUTING.md) to get started.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
