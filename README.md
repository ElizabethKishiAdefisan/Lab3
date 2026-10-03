# %% [markdown]
# LAB 3
 ### PUBH 4201 Lab 3

 %% [markdown]
 ##### Today we are using regex to parse through messy data. We are going to take our mess_data_csv and turn it into a tibble with standardization

 %%
import pandas as pd
import numpy as np
clean_our_data = pd.read_csv('messy_samples.csv')

 %% [markdown]
 ##### Our data isn't exactly the clearest. Let's use some regex coding to fix it

 %%
 We've read our data into a data frame. Lets check our sex variable
clean_our_data['sex'].value_counts()


 %% [markdown]
 #### Thats pretty confusing! We want to simplify the sexes into male, female, and other!

 %%
# Match the whole value so "female" cannot accidentally match "male".
sex = clean_our_data['sex'].astype('string').str.strip()
clean_our_data['sex'] = np.select(
    [
        sex.str.fullmatch(r'(?:m|male)', case=False, na=False),
        sex.str.fullmatch(r'(?:f|female)', case=False, na=False),
    ],
    ['Male', 'Female'],
    default='Unknown',
)

# Keep the original CSV unchanged and save the standardized data separately.
clean_our_data.to_csv('clean_our_data.csv', index=True)
clean_our_data['sex'].value_counts()

# %% [markdown]
# #### Great! now our data is easy to parse. Now, there are still more variables that are messy. Lets fix those too

# %%
# Lets check out the dob variable in our dataset
clean_our_data['dob'].value_counts()

# %%
# Standardize DOB values as MM/DD/YYYY using regex patterns
import re

month_numbers = {
    'jan': '01', 'feb': '02', 'mar': '03', 'apr': '04',
    'may': '05', 'jun': '06', 'jul': '07', 'aug': '08',
    'sep': '09', 'oct': '10', 'nov': '11', 'dec': '12',
}

def normalize_dob(value):
    if pd.isna(value):
        return value

    value = str(value).strip()
    match = re.fullmatch(r'(\d{1,2})[-/ ]+([A-Za-z]+)[-/ ,]+(\d{2,4})', value)
    if match:
        day, month_name, year = match.groups()
        month = month_numbers.get(month_name[:3].lower())
    else:
        match = re.fullmatch(r'([A-Za-z]+)[-/ ,]+(\d{1,2})[, -]+(\d{2,4})', value)
        if match:
            month_name, day, year = match.groups()
            month = month_numbers.get(month_name[:3].lower())
        else:
            match = re.fullmatch(r'(\d{4})[-/.](\d{1,2})[-/.](\d{1,2})', value)
            if match:
                year, month, day = match.groups()
            else:
                match = re.fullmatch(r'(\d{1,2})[./-](\d{1,2})[./-](\d{2,4})', value)
                if not match:
                    return value
                month, day, year = match.groups()

    if month is None:
        return value
    if len(year) == 2:
        year = ('20' if int(year) <= 25 else '19') + year

    return f'{int(month):02d}/{int(day):02d}/{year}'

clean_our_data['dob'] = clean_our_data['dob'].apply(normalize_dob)
clean_our_data.to_csv('clean_our_data.csv', index=False)
clean_our_data['dob'].value_counts()

# %% [markdown]
# #### Beautiful! it's all coming together now. Let's check the glucose variables now

# %%
# Combine the glucose_value and glucose_unit columns into a single column named 'glucose'
clean_our_data['glucose'] = clean_our_data['glucose_value'].astype(str) + ' ' + clean_our_data['glucose_unit']
clean_our_data['glucose'].value_counts()

# %% [markdown]
# #### Something is off? Some of these units are not the same as the other. To fix this, let's do some unit conversion

# %%
# Convert mmol/L values in the combined glucose column to mg/dL using regex.
import re

glucose_match = clean_our_data['glucose'].astype('string').str.extract(
    r'(?i)^\s*(?P<value>\d+(?:\.\d+)?)\s*\*?\s*mmol/L\s*$'
)
mmol_mask = glucose_match['value'].notna()
converted_values = (
    pd.to_numeric(glucose_match.loc[mmol_mask, 'value']) * 18
).map(lambda value: f'{value:.1f} mg/dL')
clean_our_data.loc[mmol_mask, 'glucose'] = converted_values
clean_our_data.to_csv('clean_our_data.csv', index=False)
clean_our_data['glucose'].value_counts()

# %%
#Now, update our file by dropping the glucose_values and glucose_unit columns.
clean_our_data = clean_our_data.drop(columns=['glucose_value', 'glucose_unit'])
clean_our_data.to_csv('clean_our_data.csv', index=False)
