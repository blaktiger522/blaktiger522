import os
import requests
from datetime import datetime, timedelta
from typing import List, Dict
import openai
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# File path to save matched jobs
SAVE_PATH = "matched_jobs.json"

# SMTP Configuration (Update with your details)
SMTP_SERVER = "smtp.example.com"
SMTP_PORT = 587
SMTP_EMAIL = "your_email@example.com"
SMTP_PASSWORD = "your_password"

# OpenAI API Key (Replace with your key)
OPENAI_API_KEY = "your_openai_api_key"
openai.api_key = OPENAI_API_KEY

# Predefined cover letter templates
TEMPLATES = {
    "tech": "As an experienced professional in technology, I have a strong background in {skills}. I am excited about the opportunity to work at {company} and contribute to {job_title}.",
    "finance": "My expertise in financial analysis and strategic planning makes me an ideal candidate for the {job_title} role at {company}.",
    "healthcare": "With a solid background in healthcare and patient management, I am eager to bring my skills to {company} as a {job_title}.",
    "education": "As a passionate educator with a commitment to learning, I am thrilled about the opportunity to join {company} as a {job_title}.",
    "marketing": "With a creative mindset and a strong understanding of marketing strategies, I am excited to bring value to {company} in the role of {job_title}.",
    "default": "I am thrilled to apply for the position of {job_title} at {company}. My skills and experiences align closely with the requirements of this role."
}

# Cover letter formats
FORMATS = {
    "formal": "This cover letter will follow a professional and structured tone.",
    "casual": "This cover letter will use a friendly and approachable tone.",
    "creative": "This cover letter will incorporate a unique and innovative style."
}

def parse_resume(file_path: str) -> str:
    """Parse content from the resume file."""
    with open(file_path, 'r') as file:
        return file.read()

def search_jobs(keywords: List[str], location: str = "", portals: List[str] = ["Indeed", "LinkedIn"]) -> List[Dict]:
    """Search jobs across portals."""
    jobs = []
    for portal in portals:
        response = requests.get(f"https://api.{portal.lower()}.com/search", params={
            "keywords": " ".join(keywords),
            "location": location
        })
        if response.status_code == 200:
            jobs += response.json().get("jobs", [])
    return jobs

def is_job_active(job: Dict) -> bool:
    """Check if the job is still active and not expired."""
    expiry_date = job.get("expiry_date")
    if expiry_date:
        expiry_date = datetime.strptime(expiry_date, "%Y-%m-%d")
        return expiry_date >= datetime.now()
    return True  # If no expiry date, assume it's active

def sort_by_expiry(jobs: List[Dict]) -> List[Dict]:
    """Sort jobs by expiry date, with soon-to-expire jobs first."""
    def get_days_to_expiry(job: Dict) -> int:
        expiry_date = job.get("expiry_date")
        if expiry_date:
            expiry_date = datetime.strptime(expiry_date, "%Y-%m-%d")
            return (expiry_date - datetime.now()).days
        return float('inf')  # Jobs without expiry date go to the bottom
    
    return sorted(jobs, key=get_days_to_expiry)

def filter_jobs(jobs: List[Dict], keywords: List[str], min_salary: int = 0, remote: bool = False) -> List[Dict]:
    """Filter jobs by relevance to keywords, activity, and advanced filters."""
    relevant_jobs = [
        job for job in jobs
        if any(keyword in job['description'].lower() for keyword in keywords)
        and is_job_active(job)
        and job.get('salary', 0) >= min_salary
        and (not remote or job.get('remote', False))  # Filter for remote jobs if specified
    ]
    return sort_by_expiry(relevant_jobs)

def generate_cover_letter_with_gpt(resume: str, job_title: str, company: str, job_description: str, user_input: str, industry: str, format_style: str) -> str:
    """Generate a personalized cover letter using OpenAI GPT."""
    template = TEMPLATES.get(industry.lower(), TEMPLATES["default"])
    format_description = FORMATS.get(format_style.lower(), FORMATS["formal"])
    prompt = f"""
    Write a {format_style} cover letter based on the following details:
    Format: {format_description}
    Template: {template}
    Candidate's resume: {resume[:300]}...
    Job title: {job_title}
    Company: {company}
    Job description: {job_description[:300]}...
    Additional details provided by the user: {user_input}
    """
    
    try:
        response = openai.Completion.create(
            engine="text-davinci-003",  # GPT model
            prompt=prompt,
            max_tokens=300,
            temperature=0.7
        )
        return response['choices'][0]['text'].strip()
    except Exception as e:
        print(f"Error generating cover letter: {e}")
        return "Error generating cover letter."

def display_jobs(jobs: List[Dict]):
    """Display jobs with application options."""
    for i, job in enumerate(jobs, start=1):
        print(f"{i}. {job['title']} at {job['company']}")
        print(f"   Location: {job['location']}")
        print(f"   Salary: {job.get('salary', 'Not specified')}")
        expiry_info = f"Expires on: {job.get('expiry_date', 'N/A')}"
        print(f"   {expiry_info}")
        print(f"   Remote: {'Yes' if job.get('remote', False) else 'No'}")
        print(f"   Link: {job['url']}")
        print("   [1] Automate Application  [2] Manual Application")

def automate_application(job_url: str, resume_path: str, cover_letter: str):
    """Automate job application if portal supports API."""
    with open(resume_path, 'rb') as resume_file:
        response = requests.post(f"{job_url}/apply", files={
            'resume': resume_file,
            'cover_letter': ('cover_letter.txt', cover_letter)
        })
    return response.status_code == 200

def main():
    # Step 1: Upload resume
    resume_path = input("Enter resume file path: ")
    if not os.path.exists(resume_path):
        print("Resume file not found!")
        return

    # Step 2: Enter search criteria
    location = input("Enter job location (optional): ")
    preferred_titles = input("Enter preferred job titles (comma-separated): ").split(",")
    min_salary = int(input("Enter minimum salary (optional, default 0): ") or 0)
    remote_only = input("Search for remote jobs only? (yes/no): ").lower() == "yes"
    additional_input = input("Enter any specific details to include in the cover letter: ")
    industry = input("Enter the industry for the job (e.g., tech, finance, healthcare, education, marketing): ").lower()
    format_style = input("Choose cover letter format (formal, casual, creative): ").lower()

    resume_content = parse_resume(resume_path)

    # Step 3: Search and filter jobs
    all_jobs = search_jobs(preferred_titles, location)
    matched_jobs = filter_jobs(all_jobs, preferred_titles, min_salary, remote_only)

    # Step 4: Save and display jobs
    with open(SAVE_PATH, "w") as file:
        file.write(str(matched_jobs))
    display_jobs(matched_jobs)

    # Step 5: Apply to jobs
    if matched_jobs:
        job_choice = int(input("Enter job number to apply: "))
        selected_job = matched_jobs[job_choice - 1]

        # Generate Cover Letter
        cover_letter = generate_cover_letter_with_gpt(
            resume_content, 
            selected_job['title'], 
            selected_job['company'], 
            selected_job.get('description', ""), 
            additional_input,
            industry,
            format_style
        )
        print("\nGenerated Cover Letter:\n")
        print(cover_letter)

        apply_option = int(input("\nEnter [1] for Automate or [2] for Manual: "))
        if apply_option == 1:
            success = automate_application(selected_job['url'], resume_path, cover_letter)
            print("Application Successful!" if success else "Application Failed!")
        elif apply_option == 2:
            print(f"Apply manually at: {selected_job['url']}")
    else:
        print("No jobs found matching your criteria.")

if __name__ == "__main__":
    main()
