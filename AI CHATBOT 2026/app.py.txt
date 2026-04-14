import streamlit as st
import pandas as pd
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

# Initialize session state
if "messages" not in st.session_state:
    st.session_state.messages = []
if "df" not in st.session_state:
    st.session_state.df = None
if "data_summary" not in st.session_state:
    st.session_state.data_summary = None

st.title("🧠 AI Table Query Chatbot (Powered by Grok)")
st.caption("Upload a CSV or Excel file and ask anything about the data!")

# Sidebar - File upload
with st.sidebar:
    st.header("Upload Table Data")
    uploaded_file = st.file_uploader("Choose CSV or Excel file", type=["csv", "xlsx", "xls"])
    
    if uploaded_file is not None:
        try:
            if uploaded_file.name.endswith(".csv"):
                df = pd.read_csv(uploaded_file)
            else:
                df = pd.read_excel(uploaded_file)
            
            st.session_state.df = df
            st.success(f"✅ Loaded {df.shape[0]} rows × {df.shape[1]} columns")
            
            # Generate data summary once
            if st.session_state.data_summary is None:
                summary = f"""**Uploaded Table Details:**
- Shape: {df.shape[0]} rows × {df.shape[1]} columns
- Columns: {list(df.columns)}
- First 5 rows:
{df.head(5).to_string(index=False)}

**Statistical Summary:**
{df.describe(include="all").to_string()}
"""
                st.session_state.data_summary = summary
            
            # Preview
            st.subheader("Data Preview")
            st.dataframe(df.head(10), use_container_width=True)
            
        except Exception as e:
            st.error(f"Error loading file: {e}")

# Main chat interface
if st.session_state.df is None:
    st.info("👆 Upload a table file to start chatting with your data!")
else:
    st.success("✅ Table loaded — ask any question below!")

    # Display chat history
    for message in st.session_state.messages:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])

    # Chat input
    if prompt := st.chat_input("Ask about the data (e.g., 'What is the total revenue?' or 'Show top 5 by sales')"):
        # Add user message
        st.session_state.messages.append({"role": "user", "content": prompt})
        with st.chat_message("user"):
            st.markdown(prompt)

        # Generate response
        with st.chat_message("assistant"):
            with st.spinner("Grok is analyzing your data..."):
                try:
                    client = OpenAI(
                        api_key=os.getenv("XAI_API_KEY"),
                        base_url="https://api.x.ai/v1",
                    )

                    # System prompt with full data context
                    system_prompt = {
                        "role": "system",
                        "content": f"""You are Grok, a highly intelligent data analyst chatbot built by xAI.
You have access to the following uploaded table data. Use it to answer accurately, perform calculations, and provide insights.

{st.session_state.data_summary}

Be concise, helpful, and always reference the actual data. If the question requires computation, do the math step-by-step."""
                    }

                    # Prepare messages for API (system + history)
                    api_messages = [system_prompt] + st.session_state.messages

                    completion = client.responses.create(
                        model="grok-4.20-reasoning",
                        input=api_messages
                    )

                    response = completion.output_text

                    st.markdown(response)
                    st.session_state.messages.append({"role": "assistant", "content": response})

                except Exception as e:
                    st.error(f"Error calling Grok API: {e}\n\nMake sure your XAI_API_KEY is set correctly.")

# Footer
st.sidebar.markdown("---")
st.sidebar.info("Built with ❤️ by Grok (xAI) • Uses Grok 4.20 Reasoning model")