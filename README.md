# Automated City-level Reporting with Generative AI and NLP

## Overview
This project dynamically generates city-level reports based on real-time socio-economic and labor market data. It integrates **NLP, Generative AI (GPT-3.5-turbo), and Neo4j graph databases** to present insights into job markets and socio-economic factors, helping businesses make data-driven decisions.

## Dataset
The dataset used in this project is provided by **Stratigens**, ensuring high-quality city-level insights into job markets and socio-economic conditions.

## Features
- **Data Extraction & Processing:** Uses Neo4j to retrieve job-related and socio-economic data.
- **Natural Language Summarization:** Applies GPT-3.5-turbo to generate human-readable summaries.
- **Dynamic Report Generation:** Utilizes FPDF to create well-structured, professional-grade PDF reports.
- **Interactive User Interface:** Built with Gradio for seamless user interaction and customized report generation.
- **Data Visualization:** Integrates Matplotlib to display job market trends.

## How It Works
1. User inputs city and job keyword via Gradio.
2. The system queries **Neo4j** for job titles, skills, and socio-economic indices.
3. The retrieved data undergoes **NLP processing** for cleaning and summarization.
4. GPT-3.5-turbo generates **custom job descriptions and city-specific insights**.
5. A **PDF report is generated** with structured insights, job trends, and charts.
6. The user downloads the report from the interface.

## Technologies Used
- **Python:** Core programming language.
- **Neo4j:** Graph database for structured job and city data storage.
- **GPT-3.5-turbo:** AI-powered text generation.
- **FPDF:** PDF generation library for structured reports.
- **Gradio:** User interface for input and report interaction.
- **Matplotlib:** Data visualization and job trend analysis.


## Example Output
A dynamically generated PDF report will include:
- **City-level socio-economic insights**
- **Job title trends and skill requirements**
- **Custom AI-generated job descriptions**
- **Graphical visualizations of job demand**

## Future Enhancements
- **Real-time data integration** from external APIs.
- **Interactive, web-based reports** instead of static PDFs.
- **Enhanced semantic search** for job retrieval using embeddings.

## Author
**Swetha Bhagavathylekshmi Elango**


