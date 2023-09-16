!pip install scikit-optimize --quiet
import  os
import numpy as np
import pandas as pd
from scipy import interp
from itertools import cycle

#PACS
#General  learning packages ( read https://scikit-learn.org)
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.svm import  SVC
from sklearn.ensemble import RandomForestClassifier
from sklearn.linear_model import SGDClassifier
from sklearn.preprocessing import MinMaxScaler #as our data convert to Byte 0-255 before are converted between 0, 1 )
from sklearn.metrics import (roc_curve,classification_report,confusion_matrix,accuracy_score, f1_score
                             , RocCurveDisplay, auc, average_precision_score,cohen_kappa_score,
                             roc_auc_score)
from skopt import BayesSearchCV
from skopt.space import Real, Categorical, Integer
from skopt.plots import plot_objective, plot_histogram
#plot data by matplotlib (  https://matplotlib.org/) and  seaborn  and Plotly
import seaborn  as  sns
import matplotlib.pyplot as plt
%matplotlib inline
plt.rcParams["figure.figsize"]=(20,16)
plt.rcParams["figure.dpi"]=150
import plotly.express as px
#filterwarnings
import warnings
warnings.filterwarnings('ignore')
plot_size = plt.rcParams["figure.figsize"]
plot_size [0] = 8
plot_size [1] = 6
plt.rcParams["figure.figsize"] = plot_size
def calculate_tpr_fpr(y_real, y_pred):
    '''
    Calculates the True Positive Rate (tpr) and the True Negative Rate (fpr) based on real and predicted observations
    
    Args:
        y_real: The list or series with the real classes
        y_pred: The list or series with the predicted classes
        
    Returns:
        tpr: The True Positive Rate of the classifier
        fpr: The False Positive Rate of the classifier
    '''
    
    # Calculates the confusion matrix and recover each element
    cm = confusion_matrix(y_real, y_pred)
    TN = cm[0, 0]
    FP = cm[0, 1]
    FN = cm[1, 0]
    TP = cm[1, 1]
    
    # Calculates tpr and fpr
    tpr =  TP/(TP + FN) # sensitivity - true positive rate
    fpr = 1 - TN/(TN+FP) # 1-specificity - false positive rate
    
    return tpr, fpr
def get_all_roc_coordinates(y_real, y_proba):
    '''
    Calculates all the ROC Curve coordinates (tpr and fpr) by considering each point as a threshold for the predicion of the class.
    
    Args:
        y_real: The list or series with the real classes.
        y_proba: The array with the probabilities for each class, obtained by using the `.predict_proba()` method.
        
    Returns:
        tpr_list: The list of TPRs representing each threshold.
        fpr_list: The list of FPRs representing each threshold.
    '''
    tpr_list = [0]
    fpr_list = [0]
    for i in range(len(y_proba)):
        threshold = y_proba[i]
        y_pred = y_proba >= threshold
        tpr, fpr = calculate_tpr_fpr(y_real, y_pred)
        tpr_list.append(tpr)
        fpr_list.append(fpr)
    return tpr_list, fpr_list

def plot_roc_curve(tpr, fpr, scatter = True, ax = None):
    '''
    Plots the ROC Curve by using the list of coordinates (tpr and fpr).
    
    Args:
        tpr: The list of TPRs representing each coordinate.
        fpr: The list of FPRs representing each coordinate.
        scatter: When True, the points used on the calculation will be plotted with the line (default = True).
    '''
    if ax == None:
        plt.figure(figsize = (5, 5))
        ax = plt.axes()
    
    if scatter:
        sns.scatterplot(x = fpr, y = tpr, ax = ax)
    sns.lineplot(x = fpr, y = tpr, ax = ax)
    sns.lineplot(x = [0, 1], y = [0, 1], color = 'green', ax = ax)
    plt.xlim(-0.05, 1.05)
    plt.ylim(-0.05, 1.05)
    plt.xlabel("False Positive Rate")
    plt.ylabel("True Positive Rate")
def ncols_calculator(cols,nrows=3):
    '''
    Takes a list of columns and numbers row plots
    
    returns number of cols to be used in matplotlib.pyplot.subplots()
    and how many axes will be remained that need to be deleted
    '''
    n = len(cols)
    ncols = n//nrows
    if ncols*nrows < n:
        ncols+=1
    axdel = ncols*nrows-n
    return ncols,axdel


def bar_matrix(df,cols,nrows=3,annot=True,title='You Forget Your Title!'):
    '''
    df --> DataFrame
    cols --> List of Columns name to be plotted
    nrows --> number of nrows to split figure subplots default is 3
    annot --> Boolean to decide whether percentage of each bar annotation is desired
    title --> Figure title to be displayed
    
    Functions is designed to create one plot using
    sns.countplot for catagorical datatype columns
    
    '''
    ncols,axdel = ncols_calculator(cols)
    fig,axes = plt.subplots(ncols,nrows,figsize=(nrows*4,ncols*3),constrained_layout=True)
    plt.suptitle(f'{title}',size=20, fontweight='bold', fontfamily='serif')
    axes=axes.ravel()
    if axdel >0:
        for ax in range(1,axdel+1):
            axes[-ax].remove()
    for i in range(len(cols)):
        #creating plotting data information
        ax = axes[i]
        col = cols[i]
        #creating plot
        sns.countplot(x=col,data=df,color=sns.color_palette()[0],ax=ax)
        #adjusting plot
        ax.set_xlabel("")
        ax.set_title(col)#+'_Column')
        ax.set_ylim(0,max(ax.get_ylim())+max(ax.get_ylim())/8)
        #writing percentage over each bar
        if annot==True:
            for p in ax.patches:
                x = p.get_x()+0.2
                y = p.get_height()+1
                percentage = '{:.1f}%'.format(100*p.get_height()/df.shape[0])
                ax.annotate(percentage,(x,y))
def hist_violin(df,cols,title='You Forget Your Title!'):
    '''
    df --> DataFrame
    cols --> List of Columns name to be plotted
    title --> Figure title to be displayed
    
    Functions is designed to create two plots using
    sns.histplot and sns.violinplot for continuous datatype columns
    
    '''
    ncols = len(cols)
    fig,axes = plt.subplots(ncols,2,figsize=(12,ncols*2),constrained_layout=True)
    fig.suptitle(f'{title}',size=20, fontweight='bold', fontfamily='serif')
    #axes=axes.ravel()
    for i in range(len(cols)):
        ax = axes[i][0]
        col = cols[i]
        sns.histplot(x=col,data=df,color=sns.color_palette()[0],ax=ax,kde=True)
        #ax.set_xlabel("")
        ax.set_title(col)
        ax = axes[i][1]
        sns.violinplot(x=col,data=df,color=sns.color_palette()[0],ax=ax)
        #to mount google drive (before  you  must have  requared data)
from google.colab import drive
drive.mount('/content/drive')
#change working directory
# من این  پوشه  در  لینک داده ارسال کردم
os.chdir("/content/drive/MyDrive/Fuzzy Classification")
#read 
df=pd.read_csv("data0.csv") # "data0.csv"
#df["target"]+=1
numeric= ['age', 'trestbps', 'chol',  'thalach', 'oldpeak']
catagoric=list(set(df.columns.values)-set(numeric))
catagoric.remove("target")
for col  in  catagoric:
  df[col]=df[col].astype(object)
catagoric
#df["target"]=[f"class{i}" for  i in df["target"].values]
#df["target"]
#taking alook on the catagorical columns
print('Catagorical Column\'s Values\n')
for col in df.select_dtypes(exclude=np.number).columns:
    unique = df[col].unique()
    print('-'*7,f'columns {col}','-'*7)
    print(f'There are {len(unique)} Unique values')
    print(f'5 of which are: {unique[:5]}\n')
    
#Numeric data statistics
print('-'*15,'\nNumeric Data Statistics\033[1;0m')
display(df.select_dtypes(include=np.number).describe())
print('-'*15,'\nPearson\'s Correlation')
plt.figure(figsize=(15,8))
sns.heatmap(df.select_dtypes(include=np.number).corr(),annot=True)
#taking alook on the catagorical columns
print('Catagorical Column\'s Values\n')
for col in df.select_dtypes(exclude=np.number).columns:
    unique = df[col].unique()
    print('-'*7,f'columns {col}','-'*7)
    print(f'There are {len(unique)} Unique values')
    print(f'5 of which are: {unique[:5]}\n')
    
#Numeric data statistics
print('-'*15,'\nNumeric Data Statistics\033[1;0m')
display(df.select_dtypes(include=np.number).describe())
print('-'*15,'\nPearson\'s Correlation')
plt.figure(figsize=(15,8))
sns.heatmap(df.select_dtypes(include=np.number).corr(),annot=True)

# select depenedt variable
y=df["target"]
#select  independent variables (reorder as shown here)
cols=[]
X=df.drop(["target"],axis=1)
X=df
#X.columns=[]
print("X",X.columns)
# split data  70%train  and 30%test
X_train, X_test, y_train, y_test = train_test_split(X, y, train_size=0.7, test_size=.3, random_state=0)
print( "smaple numbers:", "\nTrain n=:", len(X_train),"\nTest n=:",len(X_test))
y_train

# select depenedt variable
y=df["target"]
#select  independent variables (reorder as shown here)
cols=[]
X=df.drop(["target"],axis=1)
X=df
#X.columns=[]
print("X",X.columns)
# split data  70%train  and 30%test
X_train, X_test, y_train, y_test = train_test_split(X, y, train_size=0.7, test_size=.3, random_state=0)
print( "smaple numbers:", "\nTrain n=:", len(X_train),"\nTest n=:",len(X_test))
y_train
#taking alook on the catagorical columns
print('Catagorical Column\'s Values\n')
for col in df.select_dtypes(exclude=np.number).columns:
    unique = df[col].unique()
    print('-'*7,f'columns {col}','-'*7)
    print(f'There are {len(unique)} Unique values')
    print(f'5 of which are: {unique[:5]}\n')
    
#Numeric data statistics
print('-'*15,'\nNumeric Data Statistics\033[1;0m')
display(df.select_dtypes(include=np.number).describe())
print('-'*15,'\nPearson\'s Correlation')
plt.figure(figsize=(15,8))
sns.heatmap(df.select_dtypes(include=np.number).corr(),annot=True)

# select depenedt variable
y=df["target"]
#select  independent variables (reorder as shown here)
cols=[]
X=df.drop(["target"],axis=1)
X=df
#X.columns=[]
print("X",X.columns)
# split data  70%train  and 30%test
X_train, X_test, y_train, y_test = train_test_split(X, y, train_size=0.7, test_size=.3, random_state=0)
print( "smaple numbers:", "\nTrain n=:", len(X_train),"\nTest n=:",len(X_test))
y_train

df.target.value_counts().plot(kind='pie', autopct='%0.02f%%', 
                              colors=['lightblue', 'lightgreen', 'orange', 'pink'],
                              explode=(0.05, 0.05, 0.05, 0.05, 0.05))


bar_matrix(df,list(catagoric),annot=True,title='Catagorical Features Proportion')

hist_violin(df,list(numeric),title='Numerical Features Proportion')
#نمودار جعبه ای برا داده عددی numeric
for  col in  numeric:
  print(col)
  ax = df.boxplot(column=[col],by="target", showmeans=True,)
  ax.set_title(col.upper())
  ax.set_ylabel(col)
  #catagoric
for col in catagoric:
  print (col)
  dbar=df[[col,"target"]].groupby('target').value_counts().unstack()
  print(dbar)
  dbar.plot.bar()
  print ("--"*20)
  from sklearn.compose import ColumnTransformer
from sklearn.preprocessing import OneHotEncoder, StandardScaler

categorical_preprocessor = OneHotEncoder(handle_unknown="ignore")
numerical_preprocessor = StandardScaler()
preprocessor = ColumnTransformer([
    ('one-hot-encoder', categorical_preprocessor, list(catagoric)[:-1]),
    ('standard_scaler', numerical_preprocessor, list(numeric))])
from sklearn import preprocessing
#le = preprocessing.LabelEncoder().fit(y_train)
#print(le.classes_)
#y_train2=le.transform(y_train)
#y_train2

  from sklearn.neighbors import KNeighborsClassifier
# pipeline class is used as estimator to enable
# search over different model types
from sklearn.neighbors import KNeighborsClassifier
pipeKKN = Pipeline([
   ("scalar", preprocessor), ('model', KNeighborsClassifier())
])

# explicit dimension classes can be specified like this
param_dist = {
    "model__n_neighbors":Integer(1,100),
   }
knn_opt = BayesSearchCV(
    pipeKKN,
    # (parameter space, # of evaluations)
    [ (param_dist, 16)],
    cv=3
)

knn_opt.fit(X_train, y_train)

print("val. score: %s" % knn_opt.best_score_)
print("test score: %s" % knn_opt.score(X_test, y_test))
print("best params: %s" % str(knn_opt.best_params_))

d=[i.split("__")[1] for i in  knn_opt.best_params_.keys()  if  "__" in i]

_ = plot_objective(knn_opt.optimizer_results_[0],
                   dimensions=d,
                   n_minimum_search=int(1e8))
plt.tight_layout()
plt.show()
#(X_test, y_test)
from sklearn.inspection import permutation_importance
r = permutation_importance(knn_opt.best_estimator_,X_test, y_test,n_repeats=30,random_state=0)
knn_feature_importance=pd.DataFrame(r['importances_mean'], index=X_test.columns, columns=['KNN Feature Importances '])
knn_feature_importance.plot.bar()
y_pred = knn_opt.best_estimator_.predict(X_test)
y_proba = knn_opt.best_estimator_.predict_proba(X_test)
classes = knn_opt.best_estimator_.classes_

print("confusion_matrix for  KNN:\n:",confusion_matrix(y_test,y_pred))
#sns.heatmap(confusion_matrix(y_test,y_pred),annot=True)
#plt.show()
print("classification_report:\n",classification_report(y_test,y_pred))
print("cohen_kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy_score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
y_score=knn_opt.best_estimator_.predict_proba(X_test)
macro_roc_auc_ovr = roc_auc_score(y_test, y_score,    multi_class ="ovr" ,average="macro",)

print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")

y_pred = knn_opt.best_estimator_.predict(X_test)
y_proba = knn_opt.best_estimator_.predict_proba(X_test)
classes = knn_opt.best_estimator_.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("KNN calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for KNN :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))

print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")

# pipeline class is used as estimator to enable
# search over different model types
pipe = Pipeline([
    ("scalar", preprocessor),('model', SVC())
])


# single categorical value of 'model' parameter is
# sets the model class
# We will get ConvergenceWarnings because the problem is not well-conditioned.
# But that's fine, this is just an example.
linsvc_search = {
    'model': [SGDClassifier(max_iter=25,loss="modified_huber")],
    'model__penalty': ['l2', 'l1', 'elasticnet'],
    'model__l1_ratio': [0.0, 0.05, 0.1, 0.2, 0.5, 0.8, 0.9, 0.95, 1.0],
    'model__loss': ['modified_huber',
                    #'log', 'squared_hinge', 'perceptron', 'squared_loss', 'huber','epsilon_insensitive', 'squared_epsilon_insensitive'
                    ],
    'model__alpha': [10 ** x for x in range(-6, 1)],
    'model__random_state': [0]
}

# explicit dimension classes can be specified like this
svc_search = {
    'model': Categorical([SVC(max_iter=25,probability=True)]),
    'model__C':Categorical([1,50,100 ])# Real(1e-1, 1e+2, prior='log-uniform')
    ,'model__gamma': Categorical([.1,.5])#Real(1e-2, 1e+1, prior='log-uniform'),
    ,'model__degree': Integer(1,3),
    'model__kernel': Categorical(['linear', 'poly', 'rbf']),
    'model__random_state': [0]
}

opt = BayesSearchCV(
    pipe,
    # (parameter space, # of evaluations)
    [ (linsvc_search, 16), (svc_search,16)],
    cv=3
)

opt.fit(X_train, y_train)#.drop("target",axis=1)

print("val. score: %s" % opt.best_score_)
print("test score: %s" % opt.score(X_test, y_test))
print("best params: %s" % str(opt.best_params_))
d=[i.split("__")[1] for i in  opt.best_params_.keys()  if  "__" in i]
d.remove('random_state')
d
_ = plot_objective(opt.optimizer_results_[1],
                   dimensions=d,
                   n_minimum_search=int(1e8))
plt.tight_layout()
plt.show()
from sklearn.inspection import permutation_importance
r = permutation_importance(opt.best_estimator_,X_test, y_test,n_repeats=30,random_state=0)
svm_feature_importance=pd.DataFrame(r['importances_mean'], index=X_test.columns, columns=['SVM Feature Importances '])
svm_feature_importance.plot.bar()
print("SVM")
y_pred = opt.best_estimator_.predict(X_test)
y_proba = opt.best_estimator_.predict_proba(X_test)
classes = opt.best_estimator_.classes_
print("confusion_matrix:\n:",confusion_matrix(y_test,y_pred))
sns.heatmap(confusion_matrix(y_test,y_pred),annot=True)
plt.show()
print("classification_report:\n",classification_report(y_test,y_pred))
print("cohen_kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy_score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
y_score=opt.best_estimator_.predict_proba(X_test)
macro_roc_auc_ovr = roc_auc_score(y_test, y_score,multi_class="ovr",    average="macro",)
print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")

y_pred = opt.best_estimator_.predict(X_test)
y_proba = opt.best_estimator_.predict_proba(X_test)
classes = opt.best_estimator_.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("SVM calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for SVM :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))
print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))


# pipeline class is used as estimator to enable
# search over different model types
pipe2 = Pipeline([
   ("scalar", preprocessor), ('model', RandomForestClassifier(random_state=0))
])

# explicit dimension classes can be specified like this
param_dist = {
    "model__n_estimators":Integer(1,100),
    "model__max_depth":  Integer(14,20),
    #"model__max_features": Integer(0, len(cols)),
    "model__min_samples_split":Integer(2, 10), # from 2 to 10
    "model__min_samples_leaf": Integer(1, 10), # from 1 to 10
    "model__bootstrap": Categorical([True, False]), # categorical valued parameter
    "model__criterion": Categorical(["gini","entropy"]), # either "gini" or "entropy"
    'model__random_state': [0]
   }
rf_opt = BayesSearchCV(
    pipe2,
    # (parameter space, # of evaluations)
    [ (param_dist, 16)],
    cv=3
)

rf_opt.fit(X_train.drop("target",axis=1), y_train)

print("val. score: %s" % rf_opt.best_score_)
print("test score: %s" % rf_opt.score(X_test, y_test))
print("best params: %s" % str(rf_opt.best_params_))
d=[i.split("__")[1] for i in  rf_opt.best_params_.keys()  if  "__" in i]
print("dimensions",d)
if 'random_state' in d:
  d.remove('random_state')
d
_ = plot_objective(rf_opt.optimizer_results_[0],
                   dimensions=d,
                   n_minimum_search=int(1e8))
plt.tight_layout()
plt.show()
from sklearn.inspection import permutation_importance
r = permutation_importance(rf_opt.best_estimator_,X_test, y_test,n_repeats=30,random_state=0)
rf_feature_importance=pd.DataFrame(r['importances_mean'], index=X_test.columns, columns=['RF Feature Importances '])
rf_feature_importance.plot.bar()
print("RF")
y_pred = rf_opt.best_estimator_.predict(X_test)
y_proba = rf_opt.best_estimator_.predict_proba(X_test)
classes = rf_opt.best_estimator_.classes_
print("CLASSES:\n",classes)
print("confusion_matrix:\n:",confusion_matrix(y_test,y_pred))
sns.heatmap(confusion_matrix(y_test,y_pred),annot=True)
plt.show()
print("classification_report:\n",classification_report(y_test,y_pred))

print("cohen_kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy_score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
y_score=rf_opt.best_estimator_.predict_proba(X_test)
macro_roc_auc_ovr = roc_auc_score(y_test, y_score,multi_class="ovr",    average="macro",)
print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")
print("Random Forest")
y_pred = rf_opt.best_estimator_.predict(X_test)
y_proba = rf_opt.best_estimator_.predict_proba(X_test)
classes = rf_opt.best_estimator_.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("Random Forest calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for RF :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))
print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))
from sklearn.neural_network import MLPClassifier
pipe3 = Pipeline([
   ("scalar", preprocessor), ("model",MLPClassifier(solver='lbfgs', alpha=1e-5,
                  hidden_layer_sizes=(5, 2), random_state=0))
])
MLP_params = {
    "model__hidden_layer_sizes": [50, 100, 200],
    "model__activation": ["identity", "logistic", "tanh", "relu"],
    "model__solver": ["lbfgs", "sgd","adam"],
    "model__learning_rate": [ "constant", "invscaling", "adaptive"],
}
BayesNN = BayesSearchCV(pipe3, MLP_params, refit=True, cv=3, scoring='accuracy')

BayesNN.fit(X_train, y_train)


print(BayesNN.best_params_)
print("Accuracy:"+ str(BayesNN.best_score_))

print("val. score: %s" % BayesNN.best_score_)
print("test score: %s" % BayesNN.score(X_test, y_test))
print("best params: %s" % str(BayesNN.best_params_))
from sklearn.neural_network import MLPClassifier
pipe3 = Pipeline([
   ("scalar", preprocessor), ("model",MLPClassifier(solver='lbfgs', alpha=1e-5,
                  hidden_layer_sizes=(5, 2), random_state=0))
])
MLP_params = {
    "model__hidden_layer_sizes": [50, 100, 200],
    "model__activation": ["identity", "logistic", "tanh", "relu"],
    "model__solver": ["lbfgs", "sgd","adam"],
    "model__learning_rate": [ "constant", "invscaling", "adaptive"],
}
BayesNN = BayesSearchCV(pipe3, MLP_params, refit=True, cv=3, scoring='accuracy')

BayesNN.fit(X_train, y_train)


print(BayesNN.best_params_)
print("Accuracy:"+ str(BayesNN.best_score_))

print("val. score: %s" % BayesNN.best_score_)
print("test score: %s" % BayesNN.score(X_test, y_test))
print("best params: %s" % str(BayesNN.best_params_))
d=[i.split("__")[1] for i in  BayesNN.best_params_.keys()  if  "__" in i]
print("dimensions",d)
if 'random_state' in d:
  d.remove('random_state')
#d[-1]="model__optimizer__learning_rate'"

_ = plot_objective(BayesNN.optimizer_results_[0],
                   dimensions=d,
                   n_minimum_search=int(1e8))
plt.tight_layout()
plt.show()
from sklearn.inspection import permutation_importance
r = permutation_importance(BayesNN.best_estimator_,X_test, y_test,n_repeats=30,random_state=0)
nn_feature_importance=pd.DataFrame(r['importances_mean'], index=X_test.columns, columns=['NN Feature Importances '])
nn_feature_importance.plot.bar()
print("NN")
y_pred = BayesNN.best_estimator_.predict(X_test)
y_proba = BayesNN.best_estimator_.predict_proba(X_test)
classes = BayesNN.best_estimator_.classes_


print("confusion_matrix:\n:",confusion_matrix(y_test,y_pred))
sns.heatmap(confusion_matrix(y_test,y_pred),annot=True)
plt.show()
print("classification_report:\n",classification_report(y_test,y_pred))
print("cohen_kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy_score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
y_score=BayesNN.best_estimator_.predict_proba(X_test)
macro_roc_auc_ovr = roc_auc_score(y_test, y_score,multi_class="ovr",    average="macro",)
print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")
print("Neural Network")

y_pred = BayesNN.best_estimator_.predict(X_test)
y_proba = BayesNN.best_estimator_.predict_proba(X_test)
classes = BayesNN.best_estimator_.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("Neural Network calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for NN :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))
print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))

import xgboost as xgb
from scipy.stats import uniform, randint
from sklearn.ensemble import GradientBoostingClassifier
#Multiclass classification
#xgb_model 
pipe4 = Pipeline([
   ("scalar", preprocessor), ("model",GradientBoostingClassifier(random_state=42)
                              #xgb.XGBClassifier(objective="multi:softprob", random_state=42)
                              )])
gbst_params = {
    "model__loss":['log_loss', 'deviance'],
    "model__learning_rate":[0.01, .02, .03, 0.04, 0.05],
    "model__n_estimators": [50,75, 100,125, 150,175, 200],
    "model__subsample": [0.4, .5,.6,.7],
    "model__criterion":["friedman_mse", "squared_error"],
    #"model__min_samples_leaf":[0.7,0.6,0.5,0.4, 0.3,0.2],
    "model__min_weight_fraction_leaf": [0.5,0.4, 0.3,0.2],
    "model__max_features":["auto", "sqrt", "log2"],
    "model__max_depth": [2,3,4,5,6],
    #"model__ccp_alpha": [ 0.1, .2,.3,.4,0.5],
   
    
    
}


xgb_model = BayesSearchCV(pipe4, gbst_params, refit=True, cv=3, scoring='accuracy')

xgb_model.fit(X_train, y_train)


print(xgb_model.best_params_)
print("Accuracy:"+ str(BayesNN.best_score_))

print("val. score: %s" % xgb_model.best_score_)
print("test score: %s" % xgb_model.score(X_test, y_test))
print("best params: %s" % str(xgb_model.best_params_))
d=[i.split("__")[1] for i in  xgb_model.best_params_.keys()  if  "__" in i]
print("dimensions",d)
if 'random_state' in d:
  d.remove('random_state')
#d[-1]="model__optimizer__learning_rate'"

_ = plot_objective(xgb_model.optimizer_results_[0],
                   dimensions=d,
                   n_minimum_search=int(1e8)
                   )
plt.tight_layout()
plt.show()
print("XGB")
y_pred = xgb_model.best_estimator_.predict(X_test)
y_proba = xgb_model.best_estimator_.predict_proba(X_test)
classes = xgb_model.best_estimator_.classes_


print("confusion_matrix:\n:",confusion_matrix(y_test,y_pred))
sns.heatmap(confusion_matrix(y_test,y_pred),annot=True)
plt.show()
print("classification_report:\n",classification_report(y_test,y_pred))
print("cohen_kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy_score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
y_score=xgb_model.best_estimator_.predict_proba(X_test)
macro_roc_auc_ovr = roc_auc_score(y_test, y_score,multi_class="ovr",    average="macro",)
print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")
print("GBOOST")

y_pred = xgb_model.best_estimator_.predict(X_test)
y_proba = xgb_model.best_estimator_.predict_proba(X_test)
classes = xgb_model.best_estimator_.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("GBOOST calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for GB :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))
print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))

print("GBOOST")

y_pred = xgb_model.best_estimator_.predict(X_test)
y_proba = xgb_model.best_estimator_.predict_proba(X_test)
classes = xgb_model.best_estimator_.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("GBOOST calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for GB :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))
print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))
# دونلود  کدهای  داده کاوی فازی برای فازی-سایکیت
import os
os.chdir("/content")
#for info go to https://github.com/aFdezHilario/ChiFRBCSPy
!git clone  https://github.com/aFdezHilario/ChiFRBCSPy.git
os.chdir("/content/ChiFRBCSPy")
#نمایش  محل پوشه کاری
os.getcwd()

#Transform numberic features by scaling each feature to a given range. 
scaler = MinMaxScaler()
scalar=scaler.fit(X_train[numeric])
X_train[numeric]=scaler.transform(X_train[numeric])
X_test[numeric]=scaler.transform(X_test[numeric])
from sklearn import preprocessing
le = preprocessing.LabelEncoder().fit(y_train)
print(le.classes_)
y_train2=le.transform(y_train)
y_test2=le.transform(y_test)
from ChiRWClassifier import ChiRWClassifier
#All parameters are taken by default:
labels=3
tnorm="product", #This is the only available right now
rw="pcf", #Penalized Certainty Factor is the only available right now
frm="ac", #Winning Rule. "ac" (Additive Combinaton) is also avaiable

chi = ChiRWClassifier(frm=frm,labels=labels,tnorm="product")
try:
  X_test=X_test.drop("target",axis=1)
  X_train=X_train.drop("target",axis=1)
except:
  pass
chi.fit(X_train,y_train2)
y_pred = chi.predict(X_train)
print("The accuracy of Chi-FRBCS model (train) is: ", accuracy_score(y_train2,y_pred))
y_pred = chi.predict(X_test)
print("The accuracy of Chi-FRBCS model (test) is: ", accuracy_score(y_test2,y_pred))

from sklearn.inspection import permutation_importance
r = permutation_importance(chi,X_test, y_test,n_repeats=30,random_state=0)
nn_feature_importance=pd.DataFrame(r['importances_mean'], index=X_test.columns, 
                                   columns=['Chi-FRBCS Feature Importances '])
nn_feature_importance.plot.bar()
#%%capture cap
y_pred = chi.predict(X_test)
y_proba = chi.predict_proba(X_test)
classes = chi.classes_


print("confusion_matrix:\n:",confusion_matrix(y_test,y_pred))
sns.heatmap(confusion_matrix(y_test,y_pred),annot=True)
plt.show()
print("classification_report:\n",classification_report(y_test,y_pred))
print("cohen_kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy_score",accuracy_score(y_test,y_pred))
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
#y_score=chi.predict_proba(X_test)

#macro_roc_auc_ovr = roc_auc_score(y_test, y_score,multi_class="ovo",    average="macro",)
#print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")
print("Chi-FRBCS ")

y_pred = chi.predict(X_test)
y_proba = chi.predict_proba(X_test)
classes = chi.classes_
classesName=classes
print("CLASSES:\n",classes)
plt.figure(figsize = (12, 8))
bins = [i/20 for i in range(20)] + [1]
roc_auc_ovr = {}

for i in range(len(classes)):
    # Gets the class
    c = classes[i]
    
    # Prepares an auxiliar dataframe to help with the plots
    df_aux = X_test.copy()
    df_aux['class'] = [1 if y == c else 0 for y in y_test]
    df_aux['prob'] = y_proba[:, i]
    df_aux = df_aux.reset_index(drop = True)
    
    # Plots the probability distribution for the class and the rest
    ax = plt.subplot(2, 5, i+1)
    sns.histplot(x = "prob", data = df_aux, hue = 'class', color = 'b', ax = ax, bins = bins)
    ax.set_title(f"{classesName[i]}")
    ax.legend([f"Class: {classesName[i]}", "Rest"])
    ax.set_xlabel(f"P(x = {classesName[i]})")
    
    # Calculates the ROC Coordinates and plots the ROC Curves
    ax_bottom = plt.subplot(2, 5, i+6)
    tpr, fpr = get_all_roc_coordinates(df_aux['class'], df_aux['prob'])
    plot_roc_curve(tpr, fpr, scatter = False, ax = ax_bottom)
    ax_bottom.set_title("ROC Curve OvR")
    
    # Calculates the ROC AUC OvR
    roc_auc_ovr[c] = roc_auc_score(df_aux['class'], df_aux['prob'])
    
plt.tight_layout()
print("Chi-FRBCS calssifier")

avg_roc_auc = 0
i = 0
for k in roc_auc_ovr:
    avg_roc_auc += roc_auc_ovr[k]
    i += 1
    print(f"\t{classesName[k-1]} ROC AUC OvR: {roc_auc_ovr[k]:.4f}")
print(f"\naverage ROC AUC OvR: {avg_roc_auc/i:.4f}")

print("confusion matrix for Chi-FRBCS :\n:",confusion_matrix(y_test,y_pred))
print("classification report:\n",classification_report(y_test,y_pred))
print("cohen kappa_score",cohen_kappa_score(y_test,y_pred))
print("accuracy score",accuracy_score(y_test,y_pred))
macro_roc_auc_ovr=sum(roc_auc_ovr.values())/5
tpr,fpr=calculate_tpr_fpr(y_test,y_pred)
#Recall =Sensitivity = True Positive Rate=tpr
print(f"Sensitivity: {tpr}")
# FPR (1 - specificity) 
print(f"Specificity: {1-fpr}")
print(f"Macro-averaged One-vs-Rest ROC AUC score:{macro_roc_auc_ovr:.2f}")








