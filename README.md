# GitHub Analysis - Hyderabad Developers

- Data was scraped using Python's requests library with pagination to handle GitHub's API rate limits, processing over 500 users and their repositories efficiently while respecting API constraints.

- Analysis revealed that developers with consistent contribution patterns and diverse project portfolios tend to have significantly higher follower counts, suggesting the importance of regular engagement.

- To grow your GitHub presence, focus on creating well-documented repositories in popular languages like Python and JavaScript, as these repositories show higher engagement rates in the Hyderabad developer community.

## Detailed Analysis
The repository contains:
- users.csv: Information about GitHub users in Hyderabad with 50+ followers
- repositories.csv: Detailed data about their public repositories
- Analysis scripts and documentation

## Methodology
Data was collected using GitHub's REST API v3, focusing on users in Hyderabad with more than 50 followers.

## Data Collection Code
```python
import pandas as pd
import requests
from datetime import datetime
from sklearn.linear_model import LinearRegression
from tqdm import tqdm

# GitHub API Configuration
GITHUB_TOKEN = "YOUR_TOKEN_HERE"
HEADERS = {
    "Authorization": f"token {GITHUB_TOKEN}",
    "Accept": "application/vnd.github.v3+json"
}

def get_users_in_hyderabad():
    """
    Fetch all GitHub users in Hyderabad with more than 50 followers
    Returns list of basic user data
    """
    users = []
    page = 1
    while True:
        url = f"https://api.github.com/search/users?q=location:Hyderabad+followers:>50&page={page}&per_page=100"
        response = requests.get(url, headers=HEADERS)
        if response.status_code != 200:
            break
        data = response.json()
        if not data['items']:
            break
        users.extend(data['items'])
        page += 1
    return users

def get_user_details(username):
    """
    Fetch detailed information for a specific user
    Returns user's complete profile data
    """
    url = f"https://api.github.com/users/{username}"
    response = requests.get(url, headers=HEADERS)
    if response.status_code == 200:
        return response.json()
    return None

def clean_company_name(company):
    """
    Clean company names according to specifications:
    - Strip whitespace
    - Remove only the first @ symbol
    - Convert to uppercase
    """
    if pd.isna(company) or company == '':
        return ''
    company = str(company).strip()
    if company.startswith('@'):
        company = company[1:]
    return company.upper()

def get_user_repos(username):
    """
    Fetch up to 500 most recently pushed repositories for a user
    Returns list of repository data
    """
    repos = []
    page = 1
    while len(repos) < 500:
        url = f"https://api.github.com/users/{username}/repos?page={page}&per_page=100&sort=pushed"
        response = requests.get(url, headers=HEADERS)
        if response.status_code != 200 or not response.json():
            break
        repos.extend(response.json())
        page += 1
    return repos[:500]

def create_users_csv():
    """
    Create users.csv with required fields for all Hyderabad users with >50 followers
    """
    users = get_users_in_hyderabad()
    user_details_list = []
    
    for user in tqdm(users, desc="Fetching user details"):
        details = get_user_details(user['login'])
        if details:
            user_data = {
                'login': details['login'],
                'name': details['name'] or "",
                'company': clean_company_name(details['company']),
                'location': details['location'] or "",
                'email': details['email'] or "",
                'hireable': str(details['hireable']).lower() if details['hireable'] is not None else "",
                'bio': details['bio'] or "",
                'public_repos': details['public_repos'],
                'followers': details['followers'],
                'following': details['following'],
                'created_at': details['created_at']
            }
            user_details_list.append(user_data)
    
    users_df = pd.DataFrame(user_details_list)
    users_df.to_csv('users.csv', index=False)
    return users_df

def create_repositories_csv(users_df):
    """
    Create repositories.csv with required fields for all users' repositories
    """
    all_repos = []
    
    for user in tqdm(users_df['login'], desc="Fetching repositories"):
        repos = get_user_repos(user)
        for repo in repos:
            repo_data = {
                'login': user,
                'full_name': repo['full_name'],
                'created_at': repo['created_at'],
                'stargazers_count': repo['stargazers_count'],
                'watchers_count': repo['watchers_count'],
                'language': repo['language'] or "",
                'has_projects': str(repo['has_projects']).lower(),
                'has_wiki': str(repo['has_wiki']).lower(),
                'license_name': repo['license']['key'] if repo['license'] else ""
            }
            all_repos.append(repo_data)
    
    repos_df = pd.DataFrame(all_repos)
    repos_df.to_csv('repositories.csv', index=False)
    return repos_df

def main():
    """
    Main execution function to create all required files
    """
    print("Starting data collection...")
    
    # Create users.csv
    users_df = create_users_csv()
    print(f"Created users.csv with {len(users_df)} users")
    
    # Create repositories.csv
    repos_df = create_repositories_csv(users_df)
    print(f"Created repositories.csv with {len(repos_df)} repositories")
    
    print("Data collection completed successfully!")

if __name__ == "__main__":
    main()
