# Hub-utility
Hub utility


import pandas as pd
from datetime import datetime, timedelta

# -------------------------------------------------
# Helper functions
# -------------------------------------------------

def next_thursday(date):
    """
    Return the next Thursday on or after the given date
    """
    days_ahead = (3 - date.weekday()) % 7  # Monday=0 ... Thursday=3
    return date + timedelta(days=days_ahead)


def next_valid_system_date(date):
    """
    Increment system date by 1 day.
    Skip Friday, Saturday, Sunday.
    Allowed system days: Monday–Thursday only.
    """
    next_date = date + timedelta(days=1)

    # weekday(): Monday=0 ... Sunday=6
    if next_date.weekday() >= 4:  # Friday(4), Sat(5), Sun(6)
        next_date += timedelta(days=(7 - next_date.weekday()))

    return next_date


# -------------------------------------------------
# Read input files
# -------------------------------------------------

config_df = pd.read_excel("config.xlsx", sheet_name="CONFIG")

# Convert config into dictionary
config = dict(zip(config_df["Field Name"], config_df["Value"]))

calendar_start = pd.to_datetime(config["calendar_start_date"])
calendar_end = pd.to_datetime(config["calendar_end_date"])
system_date = pd.to_datetime(config["current_system_date"])

daily_start_time = config["daily_start_time"]
monthly_start_time = config["monthly_start_time"]
batch_end_time = config["batch_end_time"]
timezone = config["timezone"]

# -------------------------------------------------
# Calendar generation
# -------------------------------------------------

rows = []
cycle_no = 1

current_pointer = calendar_start

while current_pointer <= calendar_end:

    # OUR calendar batch run date (Thursday only)
    run_date = next_thursday(current_pointer)

    if run_date > calendar_end:
        break

    # HUB system dates
    before_system = system_date
    after_system = next_valid_system_date(before_system)

    # Batch type logic
    if before_system.month != after_system.month:
        batch_type = "Monthly"
        start_time = monthly_start_time
    else:
        batch_type = "Daily"
        start_time = daily_start_time

    # Append row
    rows.append({
        "Week No": run_date.isocalendar()[1],
        "Cycle No": cycle_no,
        "Before Batch - HUB Date": before_system.strftime("%A, %d-%b-%Y"),
        "After Batch - HUB Date": after_system.strftime("%A, %d-%b-%Y"),
        "HUB Batch Type": batch_type,
        "Calendar Date (Batch run date)": run_date.strftime("%A, %d-%b-%Y"),
        "Batch run start time": f"{start_time} {timezone}",
        "Batch run end time": f"{batch_end_time} {timezone}"
    })

    # Update pointers
    system_date = after_system
    cycle_no += 1
    current_pointer = run_date + timedelta(days=1)

# -------------------------------------------------
# Write output Excel
# -------------------------------------------------

output_df = pd.DataFrame(rows)
output_df.to_excel("batch_calendar.xlsx", index=False)

print("Batch calendar generated successfully: batch_calendar.xlsx")

