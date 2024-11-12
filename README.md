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

In this guide, we'll show you how to use **QuackReports** to create both static and dynamic sales reports based on a sample dataset. You'll start by creating a static report and then enhance it to be dynamic, allowing for customized reports based on user input.

### Business Scenario

Imagine you're an analyst at a retail company tasked with generating sales reports. You have sales data for different regions over several months. Initially, you need a static report summarizing overall sales. Later, you want to generate customized reports for specific regions and months dynamically.

### Sample Dataset

First, let's create a sample dataset using pandas:

```python
import pandas as pd

# Sample Sales Data
data = {
    "Region": ["North", "South", "East", "West"] * 3,
    "Month": ["January"] * 4 + ["February"] * 4 + ["March"] * 4,
    "Total Sales": [
        15000, 12000, 13000, 14000,
        16000, 11000, 15000, 13000,
        17000, 12500, 16000, 13500
    ],
    "Units Sold": [
        300, 240, 260, 280,
        320, 220, 300, 260,
        340, 250, 320, 270
    ],
    "Profit": [
        4500, 3600, 3900, 4200,
        4800, 3300, 4500, 3900,
        5100, 3750, 4800, 4050
    ]
}
df = pd.DataFrame(data)
```

Our dataframe now looks like this:

| Region | Month    | Total Sales | Units Sold | Profit |
|--------|----------|-------------|------------|--------|
| North  | January  | 15000       | 300        | 4500   |
| South  | January  | 12000       | 240        | 3600   |
| East   | January  | 13000       | 260        | 3900   |
| West   | January  | 14000       | 280        | 4200   |
| North  | February | 16000       | 320        | 4800   |
| South  | February | 11000       | 220        | 3300   |
| East   | February | 15000       | 300        | 4500   |
| West   | February | 13000       | 260        | 3900   |
| North  | March    | 17000       | 340        | 5100   |
| South  | March    | 12500       | 250        | 3750   |
| East   | March    | 16000       | 320        | 4800   |
| West   | March    | 13500       | 270        | 4050   |
    

---

### Creating a Static Report

We'll start by creating a static sales report that includes:

- **Cover Page**: Company logo and report title.
- **Sales Data Table**: Detailed sales figures.
- **Sales Trend Plot**: Visualization of total sales over months.

#### Step 1: Import Libraries

```python
import matplotlib.pyplot as plt
from quackreports import ReportEngine, Report
from quackreports.elements import Image, Text, Table, Plot
```

#### Step 2: Define Report Elements

```python
# Cover Page Elements
logo = Image("logo.png", dim=(100, 100))  # Replace with your logo path
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
```

#### Step 3: Assemble the Report Layout

```python
layout = [
    [logo, title, subtitle],  # Cover Page
    [Text("Overall Sales Data", font_size=24, bold=True), table],  # Sales Data Section
    [Text("Sales Trend", font_size=24, bold=True), plot]  # Sales Trend Section
]
```

#### Step 4: Create and Render the Report

```python
report = Report(layout)
report_engine = ReportEngine(report)
rendered_report = report_engine.run()
rendered_report.save("static_sales_report.pdf")
```
    

---

### Creating a Dynamic Report

Now, we'll enhance the report to make it dynamic, allowing customization based on user input.

#### Step 1: Define the Configuration Model

We'll use Pydantic to define a configuration schema:

```python
from pydantic import BaseModel, Field, validator

class ReportConfig(BaseModel):
    regions: list[str] = Field(..., description="List of regions to include")
    months: list[str] = Field(..., description="List of months to include")
    include_profit: bool = Field(default=False, description="Include profit data in the report")

    @validator("regions", each_item=True)
    def validate_region(cls, value):
        if value not in df["Region"].unique():
            raise ValueError(f"Invalid region: {value}")
        return value

    @validator("months", each_item=True)
    def validate_month(cls, value):
        if value not in df["Month"].unique():
            raise ValueError(f"Invalid month: {value}")
        return value
```

#### Step 2: Create Dynamic Elements

Define functions that generate report elements based on the configuration:

```python
def dynamic_title(config):
    regions = ', '.join(config.regions)
    return Text(f"Sales Report for {regions}", font_size=28, bold=True)

def dynamic_subtitle(config):
    months = ', '.join(config.months)
    return Text(f"Months: {months}", font_size=22, italic=True)

def dynamic_table(config):
    filtered_data = df[
        df["Region"].isin(config.regions) &
        df["Month"].isin(config.months)
    ]
    if not config.include_profit:
        filtered_data = filtered_data.drop(columns=["Profit"])
    return Table(filtered_data.reset_index(drop=True))
    

def dynamic_plot(config):
    filtered_data = df[
        df["Region"].isin(config.regions) &
        df["Month"].isin(config.months)
    ]
    sales_by_month = filtered_data.groupby("Month")["Total Sales"].sum()
    fig, ax = plt.subplots()
    ax.plot(sales_by_month.index, sales_by_month.values, marker='o', color='green')
    ax.set_title("Total Sales Over Selected Months")
    ax.set_xlabel("Month")
    ax.set_ylabel("Total Sales ($)")
    return Plot(fig)

def profit_summary(config):
    if config.include_profit:
        total_profit = df[
            df["Region"].isin(config.regions) &
            df["Month"].isin(config.months)
        ]["Profit"].sum()
        profit_text = Text(f"Total Profit: ${total_profit:,.2f}", font_size=18, bold=True)
        return [Text("Profit Summary", font_size=24, bold=True), profit_text]
    return []
```

#### Step 3: Assemble the Dynamic Layout

```python
def dynamic_layout(config):
    layout = [
        [logo, dynamic_title(config), dynamic_subtitle(config)],
        [Text("Filtered Sales Data", font_size=24, bold=True), dynamic_table(config)],
        [Text("Sales Trend", font_size=24, bold=True), dynamic_plot(config)]
    ]
    if config.include_profit:
        layout.append(profit_summary(config))
    return layout
```

#### Step 4: Generate Reports with Different Configurations

We can now generate both a master report and individual reports for each region:

```python
report = Report(dynamic_layout)
report_engine = ReportEngine(report, config_model=ReportConfig)

regions = list(df["Region"].unique())
months = list(df["Month"].unique())

# Generate individual reports for each region
for region in regions:
    config = ReportConfig(regions=[region], months=months, include_profit=False)
    rendered_report = report_engine.run(config)
    rendered_report.save(f"dynamic_report_{region}.pdf")

# Generate master report with all regions and profit included
master_config = ReportConfig(regions=regions, months=months, include_profit=True)
rendered_report = report_engine.run(master_config)
rendered_report.save("dynamic_report_master.pdf")
```

    

#### Step 5: Handle Invalid Configurations

Validators ensure invalid inputs are caught:

```python
# Attempt to generate a report with an invalid month
invalid_months = months + ["April"]  # Adding an invalid month

try:
    invalid_config = ReportConfig(regions=["North"], months=invalid_months, include_profit=True)
    report_engine.run(invalid_config)
except ValueError as e:
    print(f"Validation Error: {e}")
```

Output:

```
Validation Error: Invalid month: April
```

---

### Summary

By transforming a static report into a dynamic one, we've demonstrated how **QuackReports** allows you to create flexible and customizable reports. The dynamic report adjusts its content based on user-defined configurations, making it a powerful tool for generating tailored reports.

## Documentation

For detailed documentation and advanced usage, please visit our [Documentation Page](link-to-documentation).

## Contributing

We welcome contributions! Please read our [Contributing Guidelines](CONTRIBUTING.md) to get started.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
