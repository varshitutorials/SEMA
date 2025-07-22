# SEMA
v# 🚑 Smart Emergency Medical Assistance (SEMA)

## 🔍 Overview

**Smart Emergency Medical Assistance (SEMA)** is an integrated Medical SOS system designed to reduce emergency response time and improve patient outcomes. The project features three interconnected apps for **patients**, **hospitals**, and **ambulance drivers**, enabling real-time communication, live location tracking, medical history access, and automated SOS alerts.

## 📱 Platform

- **Frontend:** Flutter (Dart) for mobile apps  
- **Backend:** Node.js with Express.js  
- **Database:** Firebase Firestore / MongoDB  
- **APIs:** Google Maps, Twilio, Firebase Cloud Messaging (FCM), ImageKit  
- **Deployment:** Firebase / AWS / Heroku

---

## 🛠️ Features

### 👤 Patient App

- 📍 **Instant SOS Alerts** — Sends emergency alerts with live GPS location to hospitals and emergency contacts.
- 💊 **Medical Records Integration** — Securely shares health data like allergies, conditions, and medications with responders.
- 🌐 **Offline Mode** — Sends alerts via SMS/USSD/Bluetooth if internet is unavailable.
- 📞 **Emergency Call & SMS** — Directly contacts nearest hospital or ambulance with one tap.
- 🔔 **Real-Time Notifications** — Informs hospitals and ambulance drivers of emergencies via FCM.

### 🏥 Hospital App

- 🧭 **Live Patient Tracking** — Track location of SOS patients on a map.
- 📄 **Access Medical History** — View incoming patient's health records securely.
- 📞 **Call/Chat with Patient** — Enable voice or video communication before arrival.
- 🚑 **Ambulance Dispatch** — Assign nearest available ambulance with auto-routing.

### 🚑 Ambulance Driver App

- 🗺️ **Navigation to Patient** — Turn-by-turn Google Maps route to patient’s location.
- 🏥 **Auto-Connect to Hospital** — Displays assigned hospital and patient info.
- 🔔 **Real-Time Updates** — Push notifications for new assignments and patient vitals.

---

## 🧱 Architecture

```mermaid
graph TD
A[Patient App] -- SOS + Location --> B[Backend API]
B --> C[Twilio API - SMS/Call]
B --> D[Firebase Firestore - DB]
B --> E[Google Maps API]
B --> F[FCM - Notifications]
D --> G[Hospital App]
D --> H[Ambulance App]
