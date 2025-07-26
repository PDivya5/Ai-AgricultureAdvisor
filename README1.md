import streamlit as st
import pandas as pd
import plotly.express as px
import time
import speech_recognition as sr
import os
import google.generativeai as genai
import requests
from gtts import gTTS

# Google API Key
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY", "AIzaSyBTNjZpNhwz1rGlFuMc8CAQuCvJ-SWAOB4")
genai.configure(api_key=GEMINI_API_KEY)

# OpenWeatherMap API Key
OPENWEATHERMAP_API_KEY = "152fc02a4c31c523591c7a324d6dc3dd"

# Set background image and modern theme
st.markdown(
    """
    <style>
        .stApp {
            background-image: url("https://images.unsplash.com/photo-1530836369250-ef72a3f5cda8?ixlib=rb-1.2.1&auto=format&fit=crop&w=1950&q=80");
            background-size: cover;
            background-position: center;
            background-color: rgba(255, 255, 255, 0.9);
            background-blend-mode: lighten;
        }
        .stTitle {color: #2E8B57;}
        .stButton>button {background-color: #4CAF50; color: white; border-radius: 5px; padding: 10px 20px;}
        .stTextInput>div>div>input {border: 1px solid #4CAF50; border-radius: 5px; padding: 10px;}
        .stSlider>div>div>div>div {background-color: #4CAF50;}
        .card {background-color: rgba(255, 255, 255, 0.9); padding: 20px; border-radius: 10px; box-shadow: 0 4px 8px 0 rgba(0,0,0,0.2); margin: 10px 0;}
    </style>
    """, unsafe_allow_html=True
)


# Helper function to create cards
def create_card(title, content, color="#4CAF50"):
    st.markdown(
        f"""
        <div class="card">
            <h3 style="color: {color};">{title}</h3>
            <p>{content}</p>
        </div>
        """, unsafe_allow_html=True
    )


# --- IMPROVED SPEECH-TO-TEXT FUNCTION ---
def speech_to_text(language="English"):
    """Converts spoken language to text with better error handling."""
    language_codes = {
        "English": "en",
        "Spanish": "es",
        "French": "fr",
        "Hindi": "hi",
        "Telugu": "te"
    }
    lang_code = language_codes.get(language, "en")

    recognizer = sr.Recognizer()
    try:
        with sr.Microphone() as source:
            st.info("🎙 Speak now about your farming question...")
            recognizer.adjust_for_ambient_noise(source, duration=1)
            try:
                audio = recognizer.listen(source, timeout=7, phrase_time_limit=10)
                text = recognizer.recognize_google(audio, language=lang_code)
                return text
            except sr.WaitTimeoutError:
                return "⚠ Listening timed out. Please try again."
            except sr.UnknownValueError:
                return "❌ Could not understand the audio. Please speak clearly."
            except sr.RequestError as e:
                return f"⚠ Error with the speech service: {str(e)}"
    except Exception as e:
        return f"⚠ Microphone error: {str(e)}"


# --- TEXT-TO-SPEECH FUNCTION ---
def text_to_speech(text, language="English"):
    """Converts text into speech for agricultural advice."""
    language_codes = {
        "English": "en",
        "Spanish": "es",
        "French": "fr",
        "Hindi": "hi",
        "Telugu": "te"
    }
    lang_code = language_codes.get(language, "en")

    try:
        tts = gTTS(text=text, lang=lang_code, slow=False)
        tts.save("agri_response.mp3")
        return "agri_response.mp3"
    except Exception as e:
        st.error(f"Text-to-speech error: {str(e)}")
        return None


# Function to fetch weather data from OpenWeatherMap API
def get_weather_data(location):
    if not location:
        return None, "Please enter a valid location."

    base_url = f"http://api.openweathermap.org/data/2.5/weather?q={location}&appid={OPENWEATHERMAP_API_KEY}&units=metric"
    response = requests.get(base_url)

    if response.status_code == 200:
        data = response.json()
        return {
            "temperature": data["main"]["temp"],
            "humidity": data["main"]["humidity"],
            "weather": data["weather"][0]["description"],
            "rain_chance": data.get("rain", {}).get("1h", 0)
        }, None
    elif response.status_code == 404:
        return None, "Location not found. Please check the spelling and try again."
    elif response.status_code == 401:
        return None, "Invalid API key. Please check your OpenWeatherMap API key."
    else:
        return None, f"Failed to fetch weather data. Error: {response.status_code}"


# Function to get AI-generated responses using Google Gemini API
def get_ai_response(prompt, language="English"):
    try:
        genai.configure(api_key=GEMINI_API_KEY)
        model = genai.GenerativeModel("gemini-1.5-pro")
        response = model.generate_content(
            f"Answer the following agriculture-related question in {language}: {prompt}"
        )
        return response.text
    except Exception as e:
        return f"Error: {str(e)}"


# Front Page
def front_page():
    st.title("🌾 Welcome to the AI-Powered Agricultural Advisor")
    st.write("""
        *Revolutionizing Agriculture with AI*  
        Our platform empowers farmers with real-time insights, weather forecasts, soil analysis, and personalized crop recommendations.  
        Whether you're a small-scale farmer or managing large fields, we're here to help you optimize your farming practices.
    """)

    image_url = "https://images.unsplash.com/photo-1530836369250-ef72a3f5cda8?ixlib=rb-1.2.1&auto=format&fit=crop&w=1950&q=80"
    st.image(image_url, use_container_width=True)

    st.write("### Let's Know More About Agriculture")
    st.write("""
        Agriculture is the backbone of our society. It involves the cultivation of plants, animals, and other life forms for food, fiber, and other products used to sustain life.  
        Modern agriculture leverages technology, data, and AI to improve productivity, sustainability, and efficiency.
    """)

    if st.button("Click Here for More Details"):
        st.session_state.page = "home"


# Farm Dashboard
def farm_dashboard():
    st.title("🌱 Farm Management Dashboard")
    st.write("Welcome to your farm management dashboard. Monitor key metrics and get actionable insights.")

    col1, col2, col3 = st.columns(3)
    with col1:
        create_card("Soil Health", "Nitrogen: 60 mg/kg\nPhosphorus: 40 mg/kg\nPotassium: 50 mg/kg", "#2E8B57")
    with col2:
        create_card("Weather", "Temperature: 30°C\nHumidity: 60%\nRain Chance: 40%", "#4CAF50")
    with col3:
        create_card("Crop Health", "Pest Risk: Low\nDisease Risk: Moderate", "#FFA500")

    st.write("### Soil Nutrient Levels")
    soil_data = pd.DataFrame({
        "Nutrient": ["Nitrogen", "Phosphorus", "Potassium"],
        "Level (mg/kg)": [60, 40, 50]
    })
    fig = px.bar(soil_data, x="Nutrient", y="Level (mg/kg)", title="Soil Nutrient Levels", color="Nutrient")
    st.plotly_chart(fig)

    st.write("### Weather Trends")
    weather_data = pd.DataFrame({
        "Day": ["Mon", "Tue", "Wed", "Thu", "Fri"],
        "Temperature (°C)": [28, 30, 32, 31, 29],
        "Rain Chance (%)": [10, 20, 40, 30, 10]
    })
    fig = px.line(weather_data, x="Day", y=["Temperature (°C)", "Rain Chance (%)"], title="Weather Trends")
    st.plotly_chart(fig)


# Pest & Disease Forecasting
def pest_disease_forecasting():
    st.title("🦗 AI-Based Pest & Disease Forecasting")
    st.write("Predict potential pest or disease outbreaks based on weather and historical data.")

    location = st.text_input("Enter your location (City/Village)")
    crop = st.text_input("Enter the crop name")
    forecast_days = st.slider("Forecast for next (days)", 1, 14, 7)

    if st.button("Get Forecast"):
        with st.spinner("Fetching AI predictions..."):
            time.sleep(2)
            result = {
                "pest_risk": "High",
                "disease_risk": "Moderate",
                "suggested_measures": "Apply neem oil spray and monitor crop daily. Increase air circulation to prevent fungal diseases."
            }

            st.error(f"Pest Risk Level: {result['pest_risk']}")
            st.warning(f"Disease Risk Level: {result['disease_risk']}")
            st.info(f"Recommended Actions: {result['suggested_measures']}")


# Weather Forecasting with API Integration
def weather_forecasting():
    st.title("🌦 AI-Based Weather Forecasting")
    st.write("Get real-time weather updates to optimize farming practices.")

    location = st.text_input("Enter your location (e.g., Vijayawada,Hyderabad)")
    if st.button("Get Weather Report"):
        if not location:
            st.warning("Please enter a location.")
        else:
            with st.spinner("Fetching weather data..."):
                weather_data, error = get_weather_data(location)
                if weather_data:
                    st.success(f"Temperature: {weather_data['temperature']}°C")
                    st.info(f"Humidity: {weather_data['humidity']}%")
                    st.info(f"Weather: {weather_data['weather']}")
                    st.warning(f"Chance of Rain: {weather_data['rain_chance']}%")
                else:
                    st.error(error)


# Soil Nutrient Analysis
def soil_nutrient_analysis():
    st.title("🧪 AI-Based Soil Nutrient Analysis")
    st.write("Check the soil's nutrient levels and get fertilization recommendations.")

    nitrogen = st.number_input("Nitrogen Level (mg/kg)", 0, 100, 50)
    phosphorus = st.number_input("Phosphorus Level (mg/kg)", 0, 100, 30)
    potassium = st.number_input("Potassium Level (mg/kg)", 0, 100, 40)

    if st.button("Analyze Soil"):
        if nitrogen > 50 and phosphorus > 30 and potassium > 40:
            st.success(
                "Your soil is suitable for wheat and maize farming. Apply balanced fertilizers for optimal growth.")
        else:
            st.warning("Your soil needs improvement. Consider adding organic compost and fertilizers.")


# Crop Recommendation System
def crop_recommendation():
    st.title("🌾 Crop Recommendation System")
    st.write("Get personalized crop suggestions based on soil and weather conditions.")

    location = st.text_input("Enter your location")
    soil_type = st.selectbox("Select Soil Type", ["Loamy", "Sandy", "Clayey", "Silty"])
    if st.button("Get Crop Recommendations"):
        with st.spinner("Analyzing soil and weather data..."):
            time.sleep(2)
            if soil_type == "Loamy":
                st.success("Recommended Crops: Wheat, Maize, Soybeans")
            elif soil_type == "Sandy":
                st.success("Recommended Crops: Groundnuts, Carrots, Potatoes")
            elif soil_type == "Clayey":
                st.success("Recommended Crops: Rice, Barley, Lentils")
            elif soil_type == "Silty":
                st.success("Recommended Crops: Sunflowers, Spinach, Cabbage")


# Agriculture Tips & Videos
def agriculture_tips_videos():
    st.title("🌱 Agriculture Tips & Videos")
    st.write("Watch educational videos and learn useful tips for better farming practices.")

    st.write("### 🎥 Watch: Best Practices for Sustainable Farming")
    st.video("https://www.youtube.com/watch?v=iloAQmroRK0&t=22s")

    st.write("### 🌿 Agriculture Tips")
    tips = [
        "1. *Crop Rotation*: Rotate crops to maintain soil fertility and reduce pests.",
        "2. *Organic Fertilizers*: Use compost and manure to improve soil health.",
        "3. *Water Management*: Use drip irrigation to conserve water and reduce wastage.",
        "4. *Pest Control*: Use natural predators like ladybugs to control pests.",
        "5. *Soil Testing*: Regularly test soil to monitor nutrient levels and pH.",
        "6. *Mulching*: Apply mulch to retain soil moisture and control weeds.",
    ]
    for tip in tips:
        st.markdown(f"<div class='card'>{tip}</div>", unsafe_allow_html=True)


# AI Agriculture Questions with Text, Voice, and Weather Query
def ai_agriculture_questions():
    st.title("🤖 AI Agriculture Advisor")
    st.write("Get expert farming advice through text, voice, or weather-based queries.")

    # Language selection
    language = st.selectbox("🌍 Select Language",
                            ["English", "Spanish", "French", "Hindi", "Telugu"],
                            format_func=lambda
                                x: f"{'🇬🇧' if x == 'English' else '🇪🇸' if x == 'Spanish' else '🇫🇷' if x == 'French' else '🇮🇳' if x == 'Hindi' else '🇮🇳'} {x}")

    # Input method selection
    input_method = st.radio("Choose input method:",
                            ["✍ Text Question", "🎤 Voice Question", "🌦 Weather-Based Advice"],
                            horizontal=True)

    question = ""
    weather_data = None
    location = ""

    if input_method == "✍ Text Question":
        question = st.text_area("Type your agriculture question:",
                                placeholder="e.g., How to treat fungal infection in paddy?",
                                height=100)

    elif input_method == "🎤 Voice Question":
        if st.button("Start Recording", key="voice_button"):
            with st.spinner("Listening... Speak your farming question now"):
                voice_question = speech_to_text(language)
                if voice_question and not voice_question.startswith(("⚠", "❌")):
                    st.session_state.voice_question = voice_question
                    st.success(f"🎤 You asked: {voice_question}")
                elif voice_question.startswith(("⚠", "❌")):
                    st.warning(voice_question)

        if 'voice_question' in st.session_state:
            question = st.text_area("Your voice question",
                                    value=st.session_state.voice_question,
                                    key="voice_question_display")

    elif input_method == "🌦 Weather-Based Advice":
        location = st.text_input("📍 Enter your farm location (e.g., Vijayawada):")
        if st.button("Get Weather Advice"):
            if location:
                with st.spinner(f"Fetching weather data for {location}..."):
                    weather_data, error = get_weather_data(location)
                    if weather_data:
                        # Store weather data in session state
                        st.session_state.weather_data = weather_data
                        st.session_state.weather_location = location
                        st.success("Weather data fetched successfully!")
                    else:
                        st.error(error)
            else:
                st.warning("Please enter a location")

    # Process the question/weather query
    if st.button("Get Agricultural Advice", type="primary"):
        if input_method == "🌦 Weather-Based Advice":
            if 'weather_data' in st.session_state:
                weather_data = st.session_state.weather_data
                location = st.session_state.weather_location
                question = f"""
                Current weather conditions at {location}:
                - Temperature: {weather_data['temperature']}°C
                - Humidity: {weather_data['humidity']}%
                - Weather: {weather_data['weather']}
                - Chance of rain: {weather_data['rain_chance']}%

                Provide detailed agricultural recommendations for these conditions.
                Include specific advice for:
                1. Irrigation practices
                2. Pest/disease management
                3. Suitable crops/activities
                4. Any precautions needed
                """
            else:
                st.error("Please fetch weather data first")
                return

        if question:
            with st.spinner("Analyzing your query..."):
                try:
                    # Get AI response
                    response = get_ai_response(question, language)

                    # Display response
                    st.subheader("🌱 Agricultural Advice")

                    # Special formatting for weather-based responses
                    if input_method == "🌦 Weather-Based Advice" and weather_data:
                        st.markdown(f"""
                        <div style='background-color:#f0f8ff; padding:15px; border-radius:10px;'>
                            <h4>📍 Current Weather in {location}:</h4>
                            <p>🌡 Temperature: {weather_data['temperature']}°C<br>
                            💧 Humidity: {weather_data['humidity']}%<br>
                            ☁ Conditions: {weather_data['weather'].capitalize()}<br>
                            🌧 Rain chance: {weather_data['rain_chance']}%</p>
                            <hr>
                            <h4>Recommended Activities:</h4>
                            {response}
                        </div>
                        """, unsafe_allow_html=True)
                    else:
                        st.markdown(
                            f"<div style='background-color:#f0f8ff; padding:15px; border-radius:10px;'>{response}</div>",
                            unsafe_allow_html=True)

                    # Voice response section
                    st.markdown("---")
                    if st.checkbox("🔊 Listen to the advice", value=True, key="voice_toggle"):
                        with st.spinner("Generating voice response..."):
                            audio_file = text_to_speech(response, language)
                            if audio_file:
                                st.audio(audio_file, format="audio/mp3")
                            else:
                                st.warning("Voice response could not be generated")

                    # Additional options
                    expander = st.expander("💾 Save this advice")
                    with expander:
                        st.download_button(
                            label="Download as Text",
                            data=response,
                            file_name="agriculture_advice.txt",
                            mime="text/plain"
                        )

                except Exception as e:
                    st.error(f"⚠ Error generating response: {str(e)}")
        else:
            st.warning("Please enter or speak your question first")
# Feedback and Contact Form
def feedback_and_contact():
    st.title("📝 Feedback & Contact Us")
    st.write("We value your feedback! Please let us know how we can improve.")

    with st.form("feedback_form"):
        name = st.text_input("Name")
        email = st.text_input("Email")
        feedback = st.text_area("Your Feedback")
        submitted = st.form_submit_button("Submit")
        if submitted:
            st.success("Thank you for your feedback! We will get back to you soon.")

    st.write("### Contact Us")
    st.write("For any queries, please contact us at: *+91 9876543210*")


# Main Function
def main():
    if "page" not in st.session_state:
        st.session_state.page = "front"

    if st.session_state.page == "front":
        front_page()
    else:
        st.sidebar.title("🚜 AI-Powered Agricultural Advisor")
        st.sidebar.markdown("Choose a feature below:")
        selection = st.sidebar.radio("Navigation", [
            "🏡 Dashboard",
            "🦗 Pest & Disease Forecasting",
            "🌦 Weather Forecasting",
            "🧪 Soil Nutrient Analysis",
            "🌾 Crop Recommendation",
            "🌱 Agriculture Tips & Videos",
            "🤖 AI Agriculture Questions",
            "📝 Feedback & Contact Us"
        ])

        if selection == "🏡 Dashboard":
            farm_dashboard()
        elif selection == "🦗 Pest & Disease Forecasting":
            pest_disease_forecasting()
        elif selection == "🌦 Weather Forecasting":
            weather_forecasting()
        elif selection == "🧪 Soil Nutrient Analysis":
            soil_nutrient_analysis()
        elif selection == "🌾 Crop Recommendation":
            crop_recommendation()
        elif selection == "🌱 Agriculture Tips & Videos":
            agriculture_tips_videos()
        elif selection == "🤖 AI Agriculture Questions":
            ai_agriculture_questions()
        elif selection == "📝 Feedback & Contact Us":
            feedback_and_contact()

        st.markdown(
            """
            <div class="footer">
                <p>Powered by <strong>Navya</strong> and <strong>Divya</strong></p>
            </div>
            """, unsafe_allow_html=True
        )


if __name__ == "__main__":
    # Check for required packages
    try:
        import pyaudio
    except ImportError:
        st.warning("Installing required audio packages...")
        import subprocess
        import sys

        subprocess.check_call([sys.executable, "-m", "pip", "install", "pyaudio", "SpeechRecognition", "gTTS"])
        st.experimental_rerun()

    main()