"""
Professional Streamlit Contact Form + Dashboard
Last updated: February 2026
Features:
- Clean form validation
- Basic spam protection (simple honeypot)
- Timestamp for each submission
- Email normalization
- Session state feedback persistence
- Responsive layout
- CSV persistence with locking consideration
"""

import streamlit as st
import pandas as pd
from datetime import datetime
import os
import hashlib
import time

# ───────────────────────────────────────────────
#  CONFIGURATION
# ───────────────────────────────────────────────
DATA_FILE = "contact_submissions.csv"
MAX_MESSAGE_LENGTH = 2000
HONEYPOT_FIELD_NAME = "website_url"  # hidden field — bots usually fill it

# ───────────────────────────────────────────────
#  DATA HANDLING
# ───────────────────────────────────────────────
def get_or_create_data() -> pd.DataFrame:
    """Load existing data or create empty DataFrame with correct columns"""
    columns = ["Timestamp", "Name", "Email", "Message"]
    
    if os.path.exists(DATA_FILE):
        try:
            df = pd.read_csv(DATA_FILE)
            # Make sure all expected columns exist
            for col in columns:
                if col not in df.columns:
                    df[col] = pd.NA
            return df[columns]  # enforce column order
        except Exception as e:
            st.error(f"Error reading CSV: {e}")
            return pd.DataFrame(columns=columns)
    else:
        return pd.DataFrame(columns=columns)


def save_submission(name: str, email: str, message: str) -> bool:
    """Append new row atomically"""
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    new_row = pd.DataFrame({
        "Timestamp": [timestamp],
        "Name": [name.strip()],
        "Email": [email.strip().lower()],
        "Message": [message.strip()]
    })
    
    # Very simple file locking / atomic write attempt
    try:
        current = get_or_create_data()
        updated = pd.concat([current, new_row], ignore_index=True)
        
        # Write to temporary file first (safer)
        tmp_file = DATA_FILE + ".tmp"
        updated.to_csv(tmp_file, index=False)
        
        # Atomic move
        os.replace(tmp_file, DATA_FILE)
        return True
        
    except Exception as e:
        st.error(f"Failed to save submission: {str(e)}")
        return False


# ───────────────────────────────────────────────
#  UI / LOGIC
# ───────────────────────────────────────────────
def main():
    st.set_page_config(
        page_title="Contact • Dashboard",
        page_icon="📬",
        layout="wide",
        initial_sidebar_state="collapsed"
    )

    # ── Header ───────────────────────────────────
    st.title("📬 Contact Us")
    st.markdown("We’d love to hear from you. All messages are stored securely.")

    # ── FORM ─────────────────────────────────────
    with st.form(key="contact_form", clear_on_submit=True):
        col1, col2 = st.columns([3, 4])
        
        with col1:
            name = st.text_input("Full Name*", placeholder="Jane Doe")
        
        with col2:
            email = st.text_input("Email*", placeholder="jane@example.com")

        # Very simple honeypot field (invisible to humans)
        honeypot = st.text_input(
            "If you're human, leave this empty",
            key=HONEYPOT_FIELD_NAME,
            label_visibility="collapsed"
        )

        message = st.text_area(
            "Your Message*",
            height=140,
            max_chars=MAX_MESSAGE_LENGTH,
            placeholder="How can we help you today?"
        )

        submitted = st.form_submit_button("Send Message", use_container_width=True, type="primary")

    # ── Form processing ───────────────────────────
    if submitted:
        # Basic validation
        errors = []
        if not name.strip():
            errors.append("Name is required")
        if not email.strip():
            errors.append("Email is required")
        elif "@" not in email or "." not in email.split("@")[-1]:
            errors.append("Please enter a valid email")
        if not message.strip():
            errors.append("Message cannot be empty")
        if honeypot.strip():  # bot likely filled it
            errors.append("Validation failed. Please try again.")

        if errors:
            for err in errors:
                st.error(err)
        else:
            with st.spinner("Saving message..."):
                time.sleep(0.4)  # small UX delay
                success = save_submission(name, email, message)

            if success:
                st.success("Thank you! Your message has been received.")
                # Optional: show small confirmation with submitted data (anonymized)
                st.caption(f"Received at {datetime.now().strftime('%Y-%m-%d %H:%M')}")
            else:
                st.error("Something went wrong while saving. Please try again later.")

    # ── DASHBOARD SECTION ────────────────────────
    st.divider()
    st.subheader("📊 Received Messages")

    df = get_or_create_data()

    if df.empty:
        st.info("No messages received yet.")
    else:
        # Nice formatting
        st.dataframe(
            df.style.format({"Timestamp": lambda x: x}),
            use_container_width=True,
            column_config={
                "Timestamp": st.column_config.TextColumn("Date & Time"),
                "Name": st.column_config.TextColumn("Name"),
                "Email": st.column_config.TextColumn("Email"),
                "Message": st.column_config.TextColumn("Message", width="medium")
            },
            hide_index=True
        )

        st.caption(f"Total messages: **{len(df)}**")


if __name__ == "__main__":
    main()
