### Exploratory Data Analysis - Complete Codebase

## Get the synthetic EHR(Electronic Health Record)Data : Insrtuctions
    1. Install Java
        java --version

    2.Clone the Synthea Repository
        git clone https://github.com/synthetichealth/synthea.git
        cd synthea

    This downloads the source code to a folder called synthea.

    3.Build the Project
      ./gradlew build check --> bash
       gradlew.bat build check --> On Windows

    This will download dependencies and compile the code. It may take a few minutes.

    4. Run Synthea to Generate Data
        ./run_synthea Massachusetts -->bash
        run_synthea.bat Massachusetts --> On windows

    This will simulate synthetic patients in Massachusetts.State can be changed.
    CSV/JSON files will be generated in:synthea/output/fhir/, synthea/output/csv/, and synthea/output/ccda/
    
    5. Customize Number of Patients/Records
        Edit this in src/main/resources/synthea.properties

           generate.default_population = 1000 

        or Run
           ./run_synthea Massachusetts 500 --> bash

           This generates 500 patients

    ### Once done with the instructions, you will find files like patients.csv,conditions.csv,encounters.csv,       medications.csv etc.You may choose files according to your analysis you want to accomplish.For our use case,"procedures.csv" file has been choosen.

## load the data,get the data structure and summary of numerical attributes
    ## loading "procedures.csv" file into DataFrame:

        import pandas as pd
        df = pd.read_csv("procedures.csv") # absolute or realtive path of input csv file
        print(df.head(5))
        df.info() # For quick look at structure of data
        summary_numeric = df.describe()  # summary of each numerical attribute
        print("Numerical Summary:\n", summary_numeric)
        
    
### Non Graphical EDA:
## Univariate Non-graphical EDA
    Zero cost procedures are filterd out for clean statistics and graph plots

    ## Filter out zero-cost procedures for cleaner analysis:
        
        import pandas as pd
        df = pd.read_csv("procedures.csv")
        mask1 = (df['MAX'] == 0)
        mask2 = (df['MIN'] == 0)
        mask3 = (df['MODE'] == 0)
        combined_mask = mask1 & mask2 & mask3
        zero_value_rows = df[combined_mask]
        print(zero_value_rows) 

    ## summary of numerical attribute after filtering out zero cost procedures
        import pandas as pd
        df = pd.read_csv("procedures.csv")
        filtered_df = df[(df['MIN'] >0) & (df['MAX'] > 0) & (df['MODE'] > 0)]
        description_stats = filtered_df.describe()
        print(description_stats)

## Multivariate Non-graphical EDA
    ## Correlation matrix and correlation coefficent(range from -1 t0 +1)
       import pandas as pd
       df = pd.read_csv("procedures.csv")
       filtered_df = df[(df['MIN'] >0) & (df['MAX'] > 0) & (df['MODE'] > 0)]
       correlation_matrix=filtered_df.corr(numeric_only=True) 
       print(correlation_matrix)

    Note:numeric_only=True indicates non-numeric columns (e.g., strings, objects) in filtered_df will be ignored during the correlation calculation.But anyway our case considers only MIN,MAX,MODE numeric columns 

### Graphical EDA
## Multivariate Graphical EDA
    ## Scatter plot without Regression line:
    1.We want to visually inspect the relationships between pairs of cost columns (MIN, MODE, MAX) by drawing scatter plots, without regression lines, using a logarithmic scale to better visualize large variations in cost data.
    2.Log-log space(both axes log)is a way of transforming both axes of a plot to a logarithmic scale.
    3.log(Y) vs log(X) becomes linear equation in log-log space which gives you a straight line. Otherwise X vs Y is non linear in normal space.

        import matplotlib
        matplotlib.use('Agg')
        from matplotlib import pyplot as plt
        import pandas as pd
        import seaborn as sns
        import numpy as np

        df = pd.read_csv("procedures.csv") 

        # Set seaborn theme
        #  sns.set_theme(style="whitegrid")


        # Define column pairs for scatter plots
        pairs = [('MIN', 'MODE'), ('MIN', 'MAX'), ('MODE', 'MAX')]

        # Create subplots
        fig, axes = plt.subplots(1, 3, figsize=(18, 5))
         
        #Filters out rows where either x or y values are zero or negative, 
        #because log scales don't work with 0 or negative numbers.
        #mask is a boolean condition that selects valid rows.

        for ax, (x_col, y_col) in zip(axes, pairs):
        mask = (df[x_col] > 0) & (df[y_col] > 0)
        x = df.loc[mask, x_col]
        y = df.loc[mask, y_col]

        # Plot scatter (no regression)
        ax.scatter(x, y, alpha=0.5)

        # Set log-log scale.Converts both axes to logarithmic scale, 
        # allowing us to handle a wide range($0 -$800k) of cost values in our case effectively.
        ax.set_xscale('log')
        ax.set_yscale('log')
        ax.set_title(f'{x_col} vs {y_col} (Scatter, Log-Log)')
        ax.set_xlabel(x_col)
        ax.set_ylabel(y_col)
        plt.tight_layout()
        plt.savefig("Scatter_Plots.png")

    ## Scatter plot with Regression line:
       1.The regression line (in red) shows expected values.
       2.sns.regplot() method does not support logy as a direct keyword argument. Only logx=True is accepted.
       3.sns.regplot() applies a linear regression model in linear space, even when using logx=True.Applies log scale to x-axis which makes regression line curved.
       4.It does not fit a regression model in log-log space, just transforms the x-axis.
       5.For true linear relationship in log-log space to occur, you need to log-transform the data which we have done in this code using log10 transformation.
       6.Column pairs for scatter plot now look like log(MIN) vs log(MODE),log(MIN) vs log(MAX),log(MODE)vslog(MAX)
       7.Key difference in using code like below is that method1 changes the "visual scale of the axis" to logarithmic.But it does not "log-transform the data" before fitting the regression line.The regression line is still computed in original linear space, and just plotted on a log axis.
       8.But second method insated transforms the data into log scale (log-log transformation).Fits the linear regression on that log-transformed data.Plots the results as an actual linear relationship in log-log space.

            method 1:Fitting Regression in linear space
                sns.regplot(x='MIN', y='MAX', data=df, logx=True)
            method 2: Fitting regression in log-log space
                logX = np.log10(df['MIN'])
                logY = np.log10(df['MAX'])
                sns.regplot(x=logX, y=logY)

    ## code for Scatter plot with Regression line
           
        import matplotlib
        matplotlib.use('Agg')
        from matplotlib import pyplot as plt
        import pandas as pd
        import seaborn as sns
        import numpy as np

       df = pd.read_csv("/home/rahini/synthea/src/main/resources/costs/procedures.csv")

       # Drop non-positive values to allow log transformation
       df_filtered = df[(df[['MIN', 'MODE', 'MAX']] > 0).all(axis=1)]

       # Apply log10 transformation
       df_log = np.log10(df_filtered[['MIN', 'MODE', 'MAX']])

       # Set up subplots
       fig, axs = plt.subplots(1, 3, figsize=(18, 5))

       # Plot 1: log(MIN) vs log(MODE) ## ci=None makes plot without confidence interval for sake of simplicity.
       sns.regplot(x=df_log['MIN'], y=df_log['MODE'], ax=axs[0], scatter_kws={'alpha':0.5}, ci=None, line_kws={'color': 'red'})
       axs[0].set_title("log(MIN) vs log(MODE)")
       axs[0].set_xlabel("log10(MIN)")
       axs[0].set_ylabel("log10(MODE)")

       # Plot 2: log(MIN) vs log(MAX)
       sns.regplot(x=df_log['MIN'], y=df_log['MAX'], ax=axs[1], scatter_kws={'alpha':0.5}, ci=None, line_kws={'color': 'red'})
       axs[1].set_title("log(MIN) vs log(MAX)")
       axs[1].set_xlabel("log10(MIN)")
       axs[1].set_ylabel("log10(MAX)")

       # Plot 3: log(MODE) vs log(MAX)
       sns.regplot(x=df_log['MODE'], y=df_log['MAX'], ax=axs[2], scatter_kws={'alpha':0.5}, ci=None, line_kws={'color': 'red'})
       axs[2].set_title("log(MODE) vs log(MAX)")
       axs[2].set_xlabel("log10(MODE)")
       axs[2].set_ylabel("log10(MAX)")
       plt.tight_layout()
       plt.savefig("Scatter_Plots_with_Regression_Lines.png")

## Univariate Graphical EDA 
    ## Histogram
    ### Simple histogram for column "MAX"
        import matplotlib
        matplotlib.use('Agg')
        from matplotlib import pyplot as plt
        import pandas as pd
        import seaborn as sns
        import numpy as np
        df = pd.read_csv("procedures.csv")

        # Choose the column to plot: 'MIN', 'MODE', or 'MAX'
        column_to_plot = 'MAX'  # change to 'MIN' or 'MODE' if needed

        # Filter out zero values clearer histogram
        df_nonzero = df[df[column_to_plot] > 0]

        # Plot the histogram
        plt.figure(figsize=(10, 6))
        plt.hist(df_nonzero[column_to_plot], bins=50, edgecolor='black')
        plt.title(f'Histogram of Procedure {column_to_plot} Costs (Non-zero only)')
        plt.xlabel(f'{column_to_plot} Cost')
        plt.ylabel('Frequency')
        plt.tight_layout()
        plt.savefig("Simple_Histogram.png")

    ### Histogram for procedure without zero costs for MAX with logscale,xlim
        import matplotlib
        matplotlib.use('Agg')
        from matplotlib import pyplot as plt
        import pandas as pd
        import seaborn as sns
        import numpy as np

        df = pd.read_csv("procedures.csv")

        # Choose the column to plot: 'MIN', 'MODE', or 'MAX'
        column_to_plot = 'MAX'  # change to 'MIN' or 'MODE' if needed

        # Focus only on procedures with cost > 0 and  Zoom into a Range (e.g., under $10,000)
        df_zoomed = df[(df['MAX'] > 0) & (df['MAX'] < 10000)]

        # Plot the histogram
        plt.title(f'Histogram of Procedure {column_to_plot} Costs (Non-zero only)')
        plt.xlabel(f'{column_to_plot} Cost')
        plt.ylabel('Frequency')
        plt.tight_layout()
        plt.xscale('log')
        sns.histplot(df_zoomed['MAX'], kde=True, bins=50)
        ## For adding mean and SD lines to the graph
        mean = df_zoomed['MAX'].mean()
        std = df_zoomed['MAX'].std()
        plt.axvline(mean, color='red', linestyle='--', label=f'Mean = {mean:.2f}')
        plt.axvline(mean + std, color='green', linestyle=':', label='+1 SD')
        plt.axvline(mean - std, color='green', linestyle=':', label='-1 SD')
        plt.legend()
        plt.xlim(10, 10000) #To see mid-costs more clearly
        plt.savefig("Histogram_Procedure_without_zero_costs_Max_logscale_mean_SD")

### Histogram for Distribution of MIN,MAX,MODE procedure costs
        1.melt() method used to create long table from wide format of columns(MIN,MAX,MODE)
        ##sample data looks like:
          Cost Type      Cost
0          MIN           5935.44
1          MIN           1284.51
2          MIN           6184.83
3          MIN           5132.80
4          MIN           5132.80
       
       2. log scaled on both x,y axis as data is highly skewed(few large values and many small values)
       3. upper first SD is taken as such. but lower first SD is clipped by taking atleast small positive value from dataset using min().As costs cannot be negative.if not clipped,SD falls belew 0 which is not true for cost.

import matplotlib
matplotlib.use('Agg')
from matplotlib import pyplot as plt
import pandas as pd
import seaborn as sns
import numpy as np

df = pd.read_csv("/home/rahini/synthea/src/main/resources/costs/procedures.csv")
columns = ['MIN', 'MODE', 'MAX']

# Subplots
fig, axes = plt.subplots(1, 3, figsize=(20, 5), sharey=True)

for i, col in enumerate(columns):
    ax = axes[i]
    data = df[col]
    non_zero_data = data[data > 0]

    # Stats
    mean = non_zero_data.mean()
    std = non_zero_data.std()
    lower = max(mean - std, non_zero_data.min())
    upper = mean + std

    # Plot histogram and KDE with log scale
    sns.histplot(non_zero_data, bins=50, kde=True, ax=ax, color='skyblue', edgecolor='black', log_scale=True)

    # Draw lines
    ax.axvline(mean, color='red', linestyle='--', linewidth=2, label=f'Mean = {mean:.2f}')
    ax.axvline(lower, color='green', linestyle=':', linewidth=2, label=f'-1 SD = {lower:.2f}')
    ax.axvline(upper, color='green', linestyle=':', linewidth=2, label=f'+1 SD = {upper:.2f}')
    
    # Titles
    ax.set_title(f'{col} Cost (Non-zero)', fontsize=12)
    ax.set_xlabel('Cost (Log scale)')
    if i == 0:
        ax.set_ylabel('Frequency')
    ax.legend()

plt.tight_layout()
plt.savefig("Distribution of MIN,MODE and MAX ProcedureCosts.png")

## Box plot
    ## Box plot with log scaled and after trimming top 1% to remove extreme outliers
import matplotlib
matplotlib.use('Agg')
from matplotlib import pyplot as plt
import pandas as pd
import seaborn as sns
import numpy as np


df = pd.read_csv("procedures.csv")
min_cost = df['MIN']
non_zero_min = min_cost[min_cost > 0]

    ##log scaled
plt.boxplot(np.log10(non_zero_min), vert=True)
plt.title("Log10 Box plot for MIN (non-zero)")
plt.ylabel("Log10(MIN Cost)")
plt.grid(True)
plt.savefig("Box plot.png")

    ## Removing top 1% for outliers
threshold = non_zero_min.quantile(0.99)
filtered_min = non_zero_min[non_zero_min <= threshold]

plt.boxplot(np.log10(non_zero_min), vert=True)
plt.title("Boxplot for MIN (without top 1%) and log scaled")
plt.ylabel("Cost")
plt.grid(True)
plt.savefig("Box plot.png") 
   
## Box plot for three cost distributions on a log scale after trimming top 1% to remove extreme outliers.       
import matplotlib
matplotlib.use('Agg')
from matplotlib import pyplot as plt
import pandas as pd
import seaborn as sns
import numpy as np


df = pd.read_csv("procedures.csv")


cost_columns = ['MIN', 'MODE', 'MAX']

# Prepare subplots
fig, axs = plt.subplots(1, 3, figsize=(15, 5))

for i, col in enumerate(cost_columns):
    data = df[col]
    non_zero = data[data > 0]
    filtered = non_zero[non_zero <= non_zero.quantile(0.99)]
    
    log_data = np.log10(filtered)
    
    axs[i].boxplot(log_data)
    axs[i].set_title(f'{col} (log-scaled, <99th percentile)')
    axs[i].set_ylabel('Log10(Cost)' if i == 0 else '')
    axs[i].grid(True)

plt.tight_layout()
plt.savefig("Box plot for three cost distributions .png")
