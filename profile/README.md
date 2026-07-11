<p align="center">
  <h1 align="center">⏱ Kairos</h1>
  <p align="center">
    <strong>Taking the guesswork out of shift work.</strong>
  </p>
  <p align="center">
    A premium iOS, iPadOS &amp; watchOS app for shift workers to track rotating schedules, share them with coworkers, and never miss a shift again.
  </p>
  <p align="center">
    <a href="https://kairosapp.dev"><img src="https://img.shields.io/badge/Web-kairosapp.dev-orange?logo=safari" alt="Website" /></a>
    <img src="https://img.shields.io/badge/Swift-5.0-orange?logo=swift" alt="Swift 5.0" />
    <img src="https://img.shields.io/badge/iOS-26.5+-blue?logo=apple" alt="iOS 26.5+" />
    <img src="https://img.shields.io/badge/watchOS-Companion-green?logo=apple" alt="watchOS" />
    <img src="https://img.shields.io/badge/WidgetKit-Home%20Screen-purple?logo=apple" alt="WidgetKit" />
  </p>
</p>

---

## 📖 Overview

**Kairos** is a native Swift/SwiftUI application designed for firefighters, medical professionals, law enforcement officers, industrial workers, transportation crews, and hospitality staff who work rotating shift schedules. It provides an at-a-glance dashboard showing your current duty status, a full calendar view with day-level override controls, schedule sharing via one-time access codes, encrypted backup/recovery, and a companion watchOS app — all wrapped in a dark, premium UI with Liquid Glass effects.

---

## 🏗 Core Architecture & Repositories

The Kairos platform is divided into purpose-built repositories, each handling a specific domain:

### [`swift`](https://github.com/Kairos-LLC/.swift) — Native iOS Application
* **Purpose:** The primary mobile touchpoint for users, built specifically for the Apple ecosystem.
* **Architecture:** Written in native **Swift/SwiftUI** to deliver high performance, fluid UI/UX, and deep integration with iOS system capabilities including WidgetKit, watchOS, and Liquid Glass effects.

### [`web`](https://github.com/Kairos-LLC/.web) — Web Platform & Backend
* **Purpose:** The central hub for the [kairosapp.dev](https://kairosapp.dev) web application and backend infrastructure.
* **Infrastructure & Hosting (Vercel):** Deployed via **Vercel** for serverless edge distribution, fast load times, and seamless continuous deployment.
* **Database & Backend (Supabase):** Powered by **Supabase** (PostgreSQL) for data persistence, user authentication, and real-time data subscriptions.

### [`.github`](https://github.com/Kairos-LLC/.github) — Organization Management & Documentation
* **Purpose:** The administrative backbone housing this org profile README, project requirements, and shared configuration.

---

## ✨ Key Features

| Feature | Description |
| :--- | :--- |
| 🔥 **Dashboard** | Real-time duty status with dynamic Liquid Glass ring, countdown badge, and adaptive iPhone/iPad layout |
| 📅 **Calendar** | Full month grid with color-coded days and tap-to-override controls (vacation, extra shift, on call) |
| 🔗 **Schedule Sharing** | Generate 6-character access codes to share your schedule read-only with coworkers |
| 🔐 **Backup & Recovery** | AES-256-GCM encrypted recovery keys for full schedule restore across devices |
| 🎯 **Onboarding** | Guided setup with job role selection, 10 preset rotation patterns, and custom week builder |
| ⌚ **Apple Watch** | Companion app with progress ring, 7-day shift forecast, calendar, and settings |
| 📱 **Home Screen Widget** | Small, Medium, and Large widgets with configurable schedule selection via App Intents |

---

## 💻 Tech Stack

| Domain | Technology |
| :--- | :--- |
| **Mobile** | Native iOS/iPadOS (Swift 5, SwiftUI, Liquid Glass) |
| **Watch** | Native watchOS (SwiftUI) |
| **Widgets** | WidgetKit with App Intents |
| **Web / Frontend** | TypeScript, deployed via Vercel (Serverless/Edge) |
| **Database & Auth** | Supabase (PostgreSQL) |
| **Encryption** | Apple CryptoKit (AES-256-GCM) |
| **Version Control & CI/CD** | GitHub (Modular repository structure) |

---

## 👨‍💻 Development Team

This project is driven by **[@boazwhealy](https://github.com/boazwhealy)** (Frontend) and **[@jrdurham23](https://github.com/jrdurham23)** (Backend). The decoupled architecture allows our team to independently scale, test, and deploy the mobile app, web app, and backend services with maximum efficiency.

---

<p align="center">
  <a href="https://kairosapp.dev"><strong>kairosapp.dev</strong></a>
  <br/>
  <strong>Kairos</strong> — Taking the guesswork out of shift work.
</p>
