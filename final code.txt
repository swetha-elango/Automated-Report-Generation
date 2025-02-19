import gradio as gr
from neo4j import GraphDatabase
from fpdf import FPDF
import openai
import matplotlib.pyplot as plt
import matplotlib.ticker as ticker
import os 


# Set up OpenAI API key
openai.api_key = "secret-key"  

# Initialize Neo4j connection
password = 'Stratigens' 
database = 'neo4j'

graphDB = GraphDatabase.driver(uri='bolt://localhost:7687', auth=('neo4j', password))
session = graphDB.session(database=database)

# Utility function to sanitize user input
def sanitize_input(input_string):
    if isinstance(input_string, str):
        return input_string.strip().lower()  # Strip whitespaces and lowercase
    return input_string

def fetch_job_data(city_name, job_keyword, limit):
    city_name = sanitize_input(city_name)
    job_keyword = sanitize_input(job_keyword)
    query = """
    MATCH (s:Skill)<-[rel:TITLE_TO_SKILL]-(jt:JobTitle)-[rel2:TITLE_IN_CITY]->(ci:CityToQuery)
    WHERE toLower(jt.name) CONTAINS toLower($job_keyword)
    AND toLower(ci.cityLabel_v2) = toLower($city_name)
    WITH jt.label AS jobTitle, COLLECT(s.label) AS skills, ci.cityLabel_v2 AS city, rel2.count AS jobCount
    RETURN jobTitle, city, jobCount, skills
    LIMIT $limit
    """
    result = session.run(query, city_name=city_name, job_keyword=job_keyword, limit=limit)
    job_data = result.values()

    # Handle missing values
    cleaned_data = []
    for job in job_data:
        job_title = job[0] if job[0] else "Unknown Title"
        city = job[1] if job[1] else "Unknown City"
        job_count = job[2] if job[2] is not None else 0  # Default to 0 if job count is missing
        skills = job[3] if job[3] else ["Unknown Skills"]  # Default to "Unknown Skills" if missing
        cleaned_data.append([job_title, city, job_count, skills])
    
    return cleaned_data


def fetch_socio_economic_data(city_name):
    city_name = sanitize_input(city_name)
    query = """
    MATCH (socio:SocioeconomicIndex)-[r:INDEX_IN_CITY]->(ci:CityToQuery)
    WHERE toLower(ci.cityLabel_v2) = toLower($city_name)
    AND socio.label IN ["Cost of Living Index", "Affordability Index", "Cost of Rent Index", "Quality of Life Index", "AirQuality", "Healthcare Index", "City Population"]
    WITH COLLECT([socio.label, r.indexOriginal]) AS pairs
    RETURN apoc.map.fromPairs(pairs) AS socioEconomicIndexes
    """
    result = session.run(query, city_name=city_name)
    result_data = result.values()
    
    if result_data:
        socio_economic_data = result_data[0][0]  # Get the map structure

        # Handle missing or incomplete data
        cleaned_data = {key: (value if value is not None else "Data not available") for key, value in socio_economic_data.items()}
        return cleaned_data
    else:
        return {"Error": "No socio-economic data available for this city"}


# Use GPT-3.5-turbo to summarize socio-economic data
def summarize_socio_economic_data(socio_economic_data):
    if not socio_economic_data:
        return "No socio-economic data available."

    socio_data_str = ", ".join([f"{k}: {v}" for k, v in socio_economic_data.items()])
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": f"The following is a list of socio-economic factors: {socio_data_str}. Please summarize this data in a coherent paragraph."}
    ]

    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo", 
        messages=messages,
        max_tokens=150,
        temperature=0.7
    )
    summary = response['choices'][0]['message']['content'].strip()
    
    return summary

# Process skills list to remove duplicates and normalize
def process_skills_list(skills_list):
    return list(set([skill.strip().lower() for skill in skills_list if skill]))  # Deduplicate and clean

# Use GPT-3.5-turbo to select the top 7 suited skills
def select_top_skills_using_gpt3(job_title, skills_list):
    skills_list = process_skills_list(skills_list)  # Ensure skills are processed before sending to GPT
    skills_str = ", ".join(skills_list)
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": f"The job title is '{job_title}'. The following is a list of skills: {skills_str}.Please select the top 7 most relevant skills for this job role."}
    ]

    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=messages,
        max_tokens=100,  # Reduce max_tokens to prevent extra costs and long outputs
        temperature=0.7
    )
    
    generated_text = response['choices'][0]['message']['content'].strip()
    top_skills = [skill.strip() for skill in generated_text.split(",")[:7]]  # Limit to the top 7
    return top_skills

# Use GPT-3.5-turbo to generate job descriptions
def generate_job_description(job_title, skills_list):
    skills_str = ", ".join(skills_list)
    messages = [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": f"Write a brief job description for the role of {job_title}. The required skills for this job include: {skills_str}. Please include responsibilities, required skills, and how these skills will be applied in day-to-day work in a short paragraph."}
    ]

    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=messages,
        max_tokens=300,
        temperature=0.7
    )
    
    job_description = response['choices'][0]['message']['content'].strip()
    return job_description

# Generate a bar chart for job titles and counts
def generate_job_chart(job_data):
    job_titles = [job[0] for job in job_data]  # Job titles
    job_counts = [int(job[2]) for job in job_data]  # Job counts

    plt.figure(figsize=(8, 4))
    plt.barh(job_titles, job_counts, color='skyblue')
    plt.xlabel('Job Count')
    plt.ylabel('Job Title')
    plt.title('Job Roles and Counts')
    # Set the x-axis to show only integer values
    plt.gca().xaxis.set_major_locator(ticker.MaxNLocator(integer=True))

    chart_path = "job_chart.png"
    plt.savefig(chart_path, bbox_inches='tight')
    plt.close()

    return chart_path

# Generate a PDF report and include graphs
def generate_pdf_report(city_name, insights, jobs_data):
    pdf = FPDF()
    pdf.add_page()
    
    # Report Title
    pdf.set_font("Arial", 'B', size=16)
    pdf.cell(200, 10, txt=f"Socioeconomic Data and Jobs Report for {city_name}", ln=True, align='C')
    pdf.ln(5)
    
    # Socio-economic Summary
    pdf.set_font("Arial", 'B', size=12)
    pdf.multi_cell(0, 10, txt="Socio-Economic Summary:", align='L')
    pdf.set_font("Arial", size=11)
    pdf.multi_cell(0, 10, txt=insights[0])  # Socio-Economic Summary
    pdf.ln(5)
    
    # Job Summaries Heading
    pdf.set_font("Arial", 'B', size=14)
    pdf.cell(200, 10, txt="Jobs Summary", ln=True, align='L')
    pdf.ln(5)

    # Add each job insight
    for idx, insight in enumerate(insights[1:], 1):
        # Job Title
        pdf.set_font("Arial", 'B', size=11)
        pdf.multi_cell(0, 10, txt=f"Job Summary {idx}")
        
        # Job Details
        pdf.set_font("Arial", size=11)
        job_details = "\n".join(insight.split("\n")[1:])
        pdf.multi_cell(0, 10, txt=job_details)
        pdf.ln(5)
    
    # Generate and add the job count graph to the PDF
    graph_path = generate_job_chart (jobs_data)
    pdf.image(graph_path, x=10, y=None, w=180)  # Adjust positioning and width
    
    # Save PDF
    pdf_path = f"{city_name}_job_report.pdf"
    pdf.output(pdf_path)
    
    # Clean up the graph file to save space
    if os.path.exists(graph_path):
        os.remove(graph_path)

    return pdf_path

def is_number(value):
    try:
        float(value)  # Try to convert the value to a float
        return True
    except ValueError:
        return False

def generate_insights(data, socio_economic_data, city_name):
    insights = []
    if not data:
        return ["No job data available for the specified city and job keyword."]
    
    # Generate socio-economic summary
    socio_economic_summary = summarize_socio_economic_data(socio_economic_data)
    
    # Format the socio-economic section with indexes listed and comparison paragraph
    socio_economic_section = (
    "Indexes:\n"
    + "\n".join([f"{k}: {int(v) if k == 'City Population' and v.isdigit() else f'{float(v):.2f}' if is_number(v) and k != 'City Population' else v}" 
                 for k, v in socio_economic_data.items() if v != "Data not available"])  # Format City Population as integer and others as 2 decimal points
    + "\n\nComparison to Worldwide Cities:\n"
    f"{socio_economic_summary}"
)

    
    insights.append(socio_economic_section)
    
    # Generate insights for jobs
    for idx, job in enumerate(data, 1):
        job_title = job[0]
        all_skills = process_skills_list(job[3])  # List of all skills
        
        # Handle missing skills
        if not all_skills:
            all_skills = ["No specific skills listed"]
        
        # Use GPT-3 to select the top 7 suited skills
        top_skills = select_top_skills_using_gpt3(job_title, all_skills)
        
        job_count = job[2]
        
        # Generate job description using GPT-3
        job_description = generate_job_description(job_title, top_skills)

        # Format the final result in the new structure
        insight = (
            f"Job Summary {idx}:\n"
            f"-> Job Title: {job_title}\n"
            f"-> For the {job_title} job, there are {job_count} vacancies around the {city_name} region.\n"
            f"-> Required Skills: {', '.join(top_skills)}\n\n"
            f"-> Job Description: {job_description}\n"
        )
        
        # Append the formatted insight to the list
        insights.append(insight)
    
    return insights


# Show data button functionality
def validate_and_fetch_data(city_name, job_keyword, limit):
    if not city_name or not job_keyword:
        return "City name and job keyword are required.", None
   
    job_data = fetch_job_data(city_name, job_keyword, limit)
    socio_economic_data = fetch_socio_economic_data(city_name)
    return job_data, socio_economic_data

# Gradio interface function
def generate_report(city_name, job_keyword, limit):
    job_data, socio_economic_data = validate_and_fetch_data(city_name, job_keyword, limit)
    if not job_data or not socio_economic_data:
        return None
    insights = generate_insights(job_data, socio_economic_data, city_name)
    pdf_path = generate_pdf_report(city_name, insights, job_data)
    return pdf_path

# Gradio UI components
with gr.Blocks() as gr_interface:
    gr.Markdown("# Dynamic Socioeconomic Report Generator")
    
    with gr.Row():
        city_name = gr.Textbox(label="City Name", value="")
        job_keyword = gr.Textbox(label="Job Title Keyword / Occupation", value="")

    with gr.Row():
        limit_input = gr.Slider(label="Limit Number of Results", minimum=1, maximum=20, step=1, value=5)  # Slider for dynamic limit
    
    with gr.Row():
        show_data_button = gr.Button("Show Data")
        generate_report_button = gr.Button("Generate Report")
    
    # Display job and socio-economic data
    data_display = gr.Dataframe(headers=["Job Title", "City", "Job Count", "Skills"], interactive=False)
    socio_economic_display = gr.Textbox(label="Socio-economic Data", interactive=False)
    
    show_data_button.click(validate_and_fetch_data, inputs=[city_name, job_keyword, limit_input], outputs=[data_display, socio_economic_display])

    report_display = gr.File(label="Download Report")
    generate_report_button.click(generate_report, inputs=[city_name, job_keyword, limit_input], outputs=report_display)

gr_interface.launch()

