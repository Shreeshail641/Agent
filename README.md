# Agent
# app.py
import streamlit as st
import pandas as pd
import datetime
import os

# ----------------------------
# Mock Data Setup
# ----------------------------
# Synthetic patient database
patients = pd.DataFrame({
    "Name": ["John Doe", "Alice Smith", "Bob Lee", "Sara Khan", "Ravi Patel"],
    "DOB": ["1990-05-12", "1988-09-23", "1975-01-15", "1995-07-30", "2000-12-11"],
    "Insurance": ["Aetna", "BlueCross", "United", "Aetna", "Cigna"],
    "PatientType": ["Returning", "New", "Returning", "New", "Returning"]
})

# Doctor schedule (Excel-like simulation)
doctor_schedule = {
    "2025-09-03": ["09:00", "09:30", "10:00", "10:30"],
    "2025-09-04": ["09:00", "09:30", "10:00", "10:30"],
}

appointments = []  # store confirmed appointments

# ----------------------------
# Helper Functions
# ----------------------------
def lookup_patient(name, dob):
    match = patients[(patients["Name"].str.lower() == name.lower()) &
                     (patients["DOB"] == dob)]
    if not match.empty:
        return match.iloc[0].to_dict()
    else:
        return {"Name": name, "DOB": dob, "PatientType": "New", "Insurance": None}


def schedule_appointment(patient, date, time, insurance):
    if date in doctor_schedule and time in doctor_schedule[date]:
        doctor_schedule[date].remove(time)  # mark slot as taken
        appt = {
            "Name": patient["Name"],
            "DOB": patient["DOB"],
            "PatientType": patient["PatientType"],
            "Date": date,
            "Time": time,
            "Insurance": insurance,
            "Confirmed": True
        }
        appointments.append(appt)
        return appt
    else:
        return None


def export_appointments():
    df = pd.DataFrame(appointments)
    file = "appointments.xlsx"
    df.to_excel(file, index=False)
    return file


def send_intake_form(patient_email="patient@example.com"):
    # In real app → send via SMTP or API
    return f"Intake form sent to {patient_email}."


def send_reminder(appt, reminder_number):
    if reminder_number == 1:
        return f"Reminder {reminder_number}: Appointment on {appt['Date']} at {appt['Time']}."
    elif reminder_number == 2:
        return f"Reminder {reminder_number}: Have you filled the intake form?"
    elif reminder_number == 3:
        return f"Reminder {reminder_number}: Please confirm attendance. If cancelling, give reason."
    else:
        return None

# ----------------------------
# Streamlit UI
# ----------------------------
st.title("🩺 AI Medical Appointment Scheduler")

st.sidebar.header("Patient Information")
name = st.sidebar.text_input("Full Name")
dob = st.sidebar.date_input("Date of Birth", datetime.date(1990, 1, 1))
dob_str = dob.strftime("%Y-%m-%d")

if st.sidebar.button("Lookup Patient"):
    patient = lookup_patient(name, dob_str)
    st.session_state["patient"] = patient
    st.success(f"Patient found: {patient['PatientType']}")

if "patient" in st.session_state:
    st.write(f"**Hello {st.session_state['patient']['Name']}!**")
    st.write(f"Patient Type: {st.session_state['patient']['PatientType']}")

    # Insurance input
    insurance = st.text_input("Enter Insurance Carrier & Member ID")

    # Show available dates
    available_dates = list(doctor_schedule.keys())
    date = st.selectbox("Select Appointment Date", available_dates)

    if date:
        available_times = doctor_schedule[date]
        if available_times:
            time = st.selectbox("Select Appointment Time", available_times)
            if st.button("Book Appointment"):
                appt = schedule_appointment(st.session_state["patient"], date, time, insurance)
                if appt:
                    st.success(f"Appointment confirmed for {appt['Date']} at {appt['Time']}")
                    
                    # Export to Excel
                    file = export_appointments()
                    st.download_button("Download Admin Excel Report", open(file, "rb"), file_name=file)
                    
                    # Intake form simulation
                    form_status = send_intake_form()
                    st.info(form_status)

                    # Show reminders
                    st.subheader("Automated Reminders")
                    for i in range(1, 4):
                        st.write(send_reminder(appt, i))
                else:
                    st.error("Slot unavailable. Please choose another.")
        else:
            st.warning("No slots available for this date.")
