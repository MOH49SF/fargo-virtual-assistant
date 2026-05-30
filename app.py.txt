import streamlit as st
import google.generativeai as genai
import os

# 1. Page Configuration & Styling
st.set_page_config(page_title="Fargo Lite - Simulated Banking Assistant", page_icon="🏦", layout="centered")
st.title("🏦 Fargo Lite")
st.caption("A completely free, AI-powered virtual assistant prototype.")

# 2. Securely Retrieve the API Key
# Locally it looks for an environment variable; on Streamlit Cloud it looks in "Secrets"
api_key = os.getenv("GEMINI_API_KEY")

if not api_key:
    st.info("Please add your Gemini API key in the sidebar to test this application locally.", icon="🔑")
    user_key = st.sidebar.text_input("Enter Gemini API Key", type="password")
    if user_key:
        api_key = user_key
        genai.configure(api_key=api_key)
else:
    genai.configure(api_key=api_key)

# 3. Initialize a Stateful Mock Banking Database
if "checking_balance" not in st.session_state:
    st.session_state.checking_balance = 2450.75
if "savings_balance" not in st.session_state:
    st.session_state.savings_balance = 12800.40
if "chat_history" not in st.session_state:
    st.session_state.chat_history = []

# Display a mock dashboard widget in the sidebar so you can watch data track dynamically
st.sidebar.markdown("### 💳 Your Simulated Accounts")
st.sidebar.metric(label="Everyday Checking", value=f"${st.session_state.checking_balance:,.2f}")
st.sidebar.metric(label="Way2Save Savings", value=f"${st.session_state.savings_balance:,.2f}")
st.sidebar.markdown("---")
st.sidebar.markdown("*Note: This is an isolated mock application interface running on a free sandbox environment.*")

# 4. Construct the Guardrailed System Instructions
SYSTEM_INSTRUCTION = f"""
You are Fargo, a helpful, secure, and polite virtual assistant for Wells Fargo. 
Your tone must be professional, reassuring, clear, and concise.

You have access to the user's real-time mock balances below. Refer to them whenever they ask about their money:
- Everyday Checking Account Balance: ${st.session_state.checking_balance:.2f}
- Way2Save Savings Account Balance: ${st.session_state.savings_balance:.2f}

Capabilities:
1. You can read back account balances accurately.
2. You can answer generic questions about banking services (e.g., how to report a lost card, branch hours).

Strict Restrictions & Guardrails:
- If the user asks to transfer funds, make payments, or change data, respond with: "I can see you want to move money, but for security purposes in this prototype, transactional capabilities are disabled. Please use our secure standard menus to complete this action manually."
- Do not make up accounts or numbers.
- If the user asks non-banking, out-of-scope questions (e.g., writing code, recipes, pop culture trivia), politely steer them back to their banking needs: "I am designed specifically to assist you with your banking needs. How can I help you with your accounts today?"
"""

# 5. Build the Chat Display Engine
for message in st.session_state.chat_history:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# 6. Process User Interactions
if api_key:
    if user_input := st.chat_input("Ask Fargo about your accounts... (e.g., 'What is my checking balance?')"):
        # Display user text instantly
        with st.chat_message("user"):
            st.markdown(user_input)
        st.session_state.chat_history.append({"role": "user", "content": user_input})

        # Format chat history specifically for Gemini SDK
        formatted_history = []
        for msg in st.session_state.chat_history[:-1]:  # Exclude current message
            formatted_history.append({
                "role": "user" if msg["role"] == "user" else "model",
                "parts": [msg["content"]]
            })

        try:
            # Query the free Gemini Model
            model = genai.GenerativeModel(
                model_name="gemini-2.5-flash",
                system_instruction=SYSTEM_INSTRUCTION
            )
            
            # Initiate conversation with memory history
            chat = model.start_chat(history=formatted_history)
            response = chat.send_message(user_input)
            
            # Display Assistant Output
            with st.chat_message("assistant"):
                st.markdown(response.text)
            st.session_state.chat_history.append({"role": "assistant", "content": response.text})
            
        except Exception as e:
            st.error(f"An error occurred while connecting to the core engine: {e}")