import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.patches import Rectangle
import pingouin as pg
import requests

import requests
import pandas as pd

def load_csv_from_github(username, repo, branch, folder, file_name):
    # Construct the raw file URL to download the CSV file
    file_url = f"https://raw.githubusercontent.com/{username}/{repo}/{branch}/{folder}/{file_name}"
    file_response = requests.get(file_url)
    
    if file_response.status_code == 200:
        # Load the CSV data directly from the URL response content
        data = pd.read_csv(file_url)
        print("Data loaded successfully from GitHub!")
        return data
    else:
        print(f"Failed to download the file: {file_response.status_code}")
        return None

# Load the data
username = "MasBarr"
repo = "Barrett2024"
branch = "main"
folder = "Data/Fig4"
file_name = "Fig4_Analysis.csv"

data = load_csv_from_github(username, repo, branch, folder, file_name)

# Check if data loaded successfully before processing
if data is not None:
    # Process the loaded data
    data = data.dropna(how='all', axis=1).dropna(how='all')
    data['day'] = data['day'].astype(int)
    filtered_data = data.query('day >= 1 and day <= 12').copy()
    filtered_data['day'] = filtered_data['day'] - 1
    filtered_data['notes'] = filtered_data['notes'].str.strip().str.lower()
    filtered_data = filtered_data[~((filtered_data['notes'] == 'free with chow') & (filtered_data['day'] > 5))]
    filtered_data = filtered_data[~((filtered_data['notes'] == 'fr1 with chow') & (filtered_data['day'] == 5))]
    filtered_data = filtered_data[filtered_data['mouse'] != 'M1']
    
    average_weight_by_task = filtered_data.groupby(['day', 'notes'])['weight'].agg(['mean', 'sem']).reset_index()
    combined_avg_data = pd.concat([
        average_weight_by_task[average_weight_by_task['notes'] == 'free with chow'],
        average_weight_by_task[average_weight_by_task['notes'] == 'fr1 with chow']
    ]).sort_values(by='day').reset_index(drop=True)
    
    print("Data processing complete.")
else:
    print("Data loading failed. Check the GitHub URL or repository structure.")



# Plot settings
font_sizes = {
    'title': 8,
    'xlabel': 8,
    'ylabel': 7,
    'xticks': 6,
    'yticks': 6,
    'legend': 6,
    'inset_title': 6,
    'inset_ylabel': 6,
    'anova_text': 5
}
plt.rcParams['font.family'] = 'DejaVu Sans'
fig_width_in = 60 / 25.4
fig_height_in = 45 / 25.4

# Create the plot for Set 1
fig, ax1 = plt.subplots(figsize=(fig_width_in, fig_height_in))

# Plot for Set 1
ax1.add_patch(Rectangle((6, 20), 7.5, 15, color='red', alpha=0.1))
ax1.text(3.5, 29.4, 'Free-access', color='black', fontsize=6, ha='center')
ax1.text(8.5, 29.4, 'Fixed-Ratio 1', color='black', fontsize=6, ha='center')
x1 = combined_avg_data['day']
y1 = combined_avg_data['mean']
sem1 = combined_avg_data['sem']
ax1.plot(x1, y1, marker='o', color='black', alpha=0.2, linewidth=1.0, markersize=2)
ax1.fill_between(x1, y1 - sem1, y1 + sem1, color='grey', alpha=0.2)
ax1.spines['top'].set_visible(False)
ax1.spines['right'].set_visible(False)
ax1.set_ylim(20, 30)
ax1.set_xlim(1, 11)
ax1.set_xlabel('Experiment day', fontsize=font_sizes['xlabel'])
ax1.set_ylabel('Average Weight (Grams)', fontsize=font_sizes['ylabel'], labelpad=5)
ax1.set_xticks([2, 4, 6, 8, 10])
ax1.set_xticklabels([-4, -2, 0, 2, 4], fontsize=font_sizes['xticks'])
ax1.set_yticks([20, 25, 30])
ax1.set_yticklabels([20, 25, 30], fontsize=font_sizes['yticks'])
ax1.tick_params(axis='both', which='both', length=0)
ax1.spines['bottom'].set_color('black')
ax1.spines['left'].set_color('black')


# Adjust layout and save the figure
plt.tight_layout(pad=0)
plt.savefig("Fig.4F.pdf", format='pdf', bbox_inches='tight', pad_inches=0)
plt.show()



#STATS
# First 6 days: days 1-6
first_6_days_data = filtered_data[filtered_data['day'].between(1, 6)]

# Last 6 days: days 7-12
last_6_days_data = filtered_data[filtered_data['day'].between(7, 12)]

# Calculate average and SEM for weight during first and last 6 days
average_weight_first_6_days = first_6_days_data['weight'].mean()
sem_weight_first_6_days = first_6_days_data['weight'].sem()

average_weight_last_6_days = last_6_days_data['weight'].mean()
sem_weight_last_6_days = last_6_days_data['weight'].sem()

# Calculate average and SEM for food intake (device gain) during first and last 6 days
average_food_intake_first_6_days = first_6_days_data['device gain'].mean()
sem_food_intake_first_6_days = first_6_days_data['device gain'].sem()

average_food_intake_last_6_days = last_6_days_data['device gain'].mean()
sem_food_intake_last_6_days = last_6_days_data['device gain'].sem()

# Display the results
print(f"First 6 Days (Days 1-6):")
print(f"Average Weight: {average_weight_first_6_days:.2f} ± {sem_weight_first_6_days:.2f} grams")
print(f"Average Food Intake: {average_food_intake_first_6_days:.2f} ± {sem_food_intake_first_6_days:.2f} grams")

print(f"\nLast 6 Days (Days 7-12):")
print(f"Average Weight: {average_weight_last_6_days:.2f} ± {sem_weight_last_6_days:.2f} grams")
print(f"Average Food Intake: {average_food_intake_last_6_days:.2f} ± {sem_food_intake_last_6_days:.2f} grams")


# Prepare the data for the paired t-test using Pingouin

# For weight: Average weight by mouse for first and last 6 days
weight_first_6_days = first_6_days_data.groupby('mouse')['weight'].mean().reset_index(name='weight_first')
weight_last_6_days = last_6_days_data.groupby('mouse')['weight'].mean().reset_index(name='weight_last')

# Merge the first and last weights into one dataframe
weight_data = weight_first_6_days.merge(weight_last_6_days, on='mouse')

# Perform paired t-test for weight
weight_ttest = pg.ttest(weight_data['weight_first'], weight_data['weight_last'], paired=True)

# For food intake: Average intake by mouse for first and last 6 days
intake_first_6_days = first_6_days_data.groupby('mouse')['device gain'].mean().reset_index(name='intake_first')
intake_last_6_days = last_6_days_data.groupby('mouse')['device gain'].mean().reset_index(name='intake_last')

# Merge the first and last food intake into one dataframe
intake_data = intake_first_6_days.merge(intake_last_6_days, on='mouse')

# Perform paired t-test for food intake
intake_ttest = pg.ttest(intake_data['intake_first'], intake_data['intake_last'], paired=True)

# Display the results
print("Paired t-test for Weight (First 6 Days vs. Last 6 Days):")
print(weight_ttest)

print("\nPaired t-test for Food Intake (First 6 Days vs. Last 6 Days):")
print(intake_ttest)
