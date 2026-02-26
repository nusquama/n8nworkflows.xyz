Validate email font families from Gmail with Google Sheets

https://n8nworkflows.xyz/workflows/validate-email-font-families-from-gmail-with-google-sheets-13566


# Validate email font families from Gmail with Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Validate the **font-family used in specific pieces of email content** by extracting “actual” font-family values from **Gmail message HTML**, then comparing them to **expected** font-family values stored in **Google Sheets**, and finally writing pass/fail results back to Google Sheets.

**Target use cases:**
- Email QA for brand compliance (headings, body, CTA labels, footer/disclaimer text).
- Automated validation of typography across marketing email templates.

### 1.1 Execution & Email Retrieval
Manual execution triggers a Gmail query that pulls unread messages from a specific sender.

### 1.2 Expected Specs Loading (Google Sheets)
Loads a