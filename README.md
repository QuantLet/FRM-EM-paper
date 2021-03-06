[<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/banner.png" width="888" alt="Visit QuantNet">](http://quantlet.de/)

## [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/qloqo.png" alt="Visit QuantNet">](http://quantlet.de/) **FRM-EM-paper** [<img src="https://github.com/QuantLet/Styleguide-and-FAQ/blob/master/pictures/QN2.png" width="60" alt="Visit QuantNet 2.0">](http://quantlet.de/)

```yaml


Name of Quantlet: FRM-EM-paper

Published in: ''

Description: 'Uplifted hierarchical risk parity based FRM portfolio construction.'

Keywords: 'FRM (Financial Risk Meter), Emerging Markets, Uplifted Hierarchical Risk Parity, Inverse lambda, Min Variance'

Author: 'Souhir Ben Amor'

See also: ''

Submitted:  '08.02.2021'

Additional Info : 'This repository implements the upHRP algorithm, the MinVAr portfolio, and the InvLamda portfolio.
The repository includes a code for the classical portfolio optimisation approaches (MinVar, IV, and HRP),
and another code for the Uplifted portfolio optimisation approaches upHRP and the InvLambda. 
The input data, as well as the output, are also provided.
Code for classical HRP is based on Lopez de Prado, M. (2018). Advances in Financial 
Machine Learning. Wiley. The code has been modified to create an uplifted portfolio
strategies based on FRM (Financial Risk Meter) adjacency matrices and it is adapted to be used with
python 3 and the data set.'

```

![Picture1](Backtesting%20modern%20approach%20with%20different%20tau%20.png)

![Picture2](Corr_Heatmap_EM_Cla.HRP_sorted.png)

![Picture3](Corr_Heatmap_EM_Cla.HRP_unsorted.png)

![Picture4](Corr_Heatmap_EM_m.HRP_sorted%200505.png)

![Picture5](Corr_Heatmap_EM_m.HRP_unsorted%200505.png)

![Picture6](Dendrogram_EM_M.HRP_20200630.png)

![Picture7](Dendrogram_FIs_010.png)

![Picture8](Dendrogram_FIs_025.png)

![Picture9](Dendrogram_FIs_050png.png)

![Picture10](Dendrogram__EM_Cla.HRP.png)

![Picture11](FRM%202020.png)

![Picture12](Portfolio%20returns%2030%20days%20Rebalacing.png)

![Picture13](lamda_weights%2020200630-4.png)

### PYTHON Code
```python

#This file implements the upHRP algorithm, the MinVAr portfolio and the InvLamda portf.
#Code for classical HRP is based on Lopez de Prado, M. (2018). Advances in Financial 
#Machine Learning. Wiley. The code has been modified to create an uplifted portfolio
#strategies based on FRM adjacency mtrices and its adapted in order to be used with
#python 3 and the data set.
 
#Souhir Ben Amor
#@date: 20201010
#"""

#[0] Import library
import numpy as np
import matplotlib.pyplot as plt
import pandas as pd

#[0]Upload input data, Financial Institutions of the 6 Emerging Markets and adjacency matrix
FIs_prices = pd.read_excel("Financial Institutions Price Series.xlsx")
FRM_EM_Adjacency_matrix= pd.read_csv("adj_matix_20200630_050.csv")
print(FIs_prices)

# In[1]: 
# Load modules

import os
path = os.getcwd() # Set Working directory here

# Import modules for Datastructuring and calc.
import pandas as pd
import numpy as np
from scipy import stats
import warnings
from tqdm import tqdm

# Modules for RHP algorithm
import matplotlib.pyplot as mpl
import scipy.cluster.hierarchy as sch

# Modules for the network plot
import networkx as nx
from networkx.convert_matrix import from_numpy_matrix

# Modules for Markowitz optimization
import cvxopt as opt
import cvxopt.solvers as optsolvers

warnings.filterwarnings("ignore") # suppress warnings in clustering

# In[2]: 
# define functions for HRP and IVP

def getIVP(cov,**kargs):
    # Compute the inverse-variance portfolio
    ivp=1./np.diag(cov)
    ivp/=ivp.sum()
    return ivp


def getClusterVar(cov, cItems):
    # Compute variance per cluster
    cov_=cov.loc[cItems, cItems] # matrix slice
    w_=getIVP(cov_).reshape(-1,1)
    cVar=np.dot(np.dot(w_.T,cov_),w_)[0,0]
    return cVar


def getQuasiDiag(link):
    # Sort clustered items by distance
    link=link.astype(int)
    sortIx=pd.Series([link[-1,0],link[-1,1]])
    numItems=link[-1,3] # number of original items
    while sortIx.max() >=numItems:
        sortIx.index=range(0,sortIx.shape[0]*2,2) # make space
        df0=sortIx[sortIx>=numItems] # find clusters
        i=df0.index;j=df0.values-numItems
        sortIx[i]=link[j,0] # item 1
        df0=pd.Series(link[j,1], index=i+1)
        sortIx=sortIx.append(df0) # item 2
        sortIx=sortIx.sort_index() # re-sort
        sortIx.index=range(sortIx.shape[0]) # re-index
    return sortIx.tolist()


def getRecBipart(cov,sortIx):
    # Compute HRP alloc
    w=pd.Series(1,index=sortIx)
    cItems=[sortIx] # initialize all items in one cluster
    while len(cItems)>0:
        cItems=[i[j:k] for i in cItems for j,k in ((0,len(i)//2),(len(i)//2,\
                len(i))) if len(i)>1] # bi-section
        for i in range(0,len(cItems),2): # parse in pairs
            cItems0=cItems[i] # cluster 1
            cItems1=cItems[i+1] # cluster 2
            cVar0=getClusterVar(cov,cItems0)
            cVar1=getClusterVar(cov,cItems1)
            alpha=1-cVar0/(cVar0+cVar1)
            w[cItems0]*=alpha # weight 1
            w[cItems1]*=1-alpha # weight 2
    return w


def correlDist(corr):
    # A distance matrix based on correlation, where 0<=d[i,j]<=1 
    # This is a proper distance metric
    dist=((1-corr)/2.)**.5 # distance matrix
    return dist


def plotCorrMatrix(path, corr, labels=None):
    # Heatmap of the correlation matrix
    if labels is None:labels=[]
    mpl.pcolor(corr)
    mpl.colorbar()
    mpl.yticks(np.arange(.5,corr.shape[0]+.5),labels)
    mpl.xticks(np.arange(.5,corr.shape[0]+.5),labels)
    mpl.savefig(path,dpi=300, transparent=True)
    mpl.clf();mpl.close() # reset pylab
    return

# In[3]: 
# define function for MinVar portfolio
    
# The MIT License (MIT)
#
# Copyright (c) 2015 Christian Zielinski
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.   

def min_var_portfolio(cov_mat, allow_short=False):
    """
    Computes the minimum variance portfolio.

    Note: As the variance is not invariant with respect
    to leverage, it is not possible to construct non-trivial
    market neutral minimum variance portfolios. This is because
    the variance approaches zero with decreasing leverage,
    i.e. the market neutral portfolio with minimum variance
    is not invested at all.
    
    Parameters
    ----------
    cov_mat: pandas.DataFrame
        Covariance matrix of asset returns.
    allow_short: bool, optional
        If 'False' construct a long-only portfolio.
        If 'True' allow shorting, i.e. negative weights.

    Returns
    -------
    weights: pandas.Series
        Optimal asset weights.
    """
    if not isinstance(cov_mat, pd.DataFrame):
        raise ValueError("Covariance matrix is not a DataFrame")

    n = len(cov_mat)
        
    P = opt.matrix(cov_mat.values)
    q = opt.matrix(0.0, (n, 1))

# Constraints Gx <= h
    if not allow_short:
    # x >= 0
       G = opt.matrix(-np.identity(n))
       h = opt.matrix(0.0, (n, 1))
    else:
        G = None
        h = None

# Constraints Ax = b
# sum(x) = 1
    A = opt.matrix(1.0, (1, n))
    b = opt.matrix(1.0)

    # Solve
    optsolvers.options['show_progress'] = False
    sol = optsolvers.qp(P, q, G, h, A, b)
        
    if sol['status'] != 'optimal':
        warnings.warn("Convergence problem")
            
 # Put weights into a labeled series
    weights = pd.Series(sol['x'], index=cov_mat.index)
    return weights  

# In[4]: 
# Define functions for network graphs
    
#Function to plot Network plots
def plotNetwork(path,corr):
    # Transform it in a links data frame
    #links=corr.stack().reset_index()
    #Build graph
    corr=Corr_mat
    adj_matrix = corr
    constits_latest = corr.index
    # remove self-loops
    adj_matrix = np.where((adj_matrix<=1.000001) & (adj_matrix>=0.99999),0,adj_matrix)
    # replace values that are below threshold
    # create undirected graph from adj_matrix
    graph = from_numpy_matrix(adj_matrix, parallel_edges=False, create_using= nx.Graph())
    # set names to crypots
    graph = nx.relabel.relabel_nodes(graph, dict(zip(range(len(constits_latest)), constits_latest)))
    pos_og =  nx.circular_layout(graph, scale=2)
    pos = nx.circular_layout(graph, scale=1.7)
    
    for p in pos:  # raise text positions
        if pos[p][1]>1:
            pos[p][1] += 0.15
        if pos[p][1]<-1:
            pos[p][1] -= 0.15
        elif pos[p][0]<0:
            pos[p][0] -= 0.3
        else:
            pos[p][0]+=0.3
    plt = mpl.figure(figsize = (5,5)) 
    nx.draw(graph, pos_og, with_labels= False)
    nx.draw_networkx_labels(graph, pos)
     
    plt.savefig(path,dpi=300 ,transparent=True)
    mpl.clf();mpl.close()
    return

## In[5]:
# Loading and structuring crypto data sets

FIs_prices = FIs_prices[(~FIs_prices.isnull()).all(axis=1)] # Deleting empty rows
FIs_prices = FIs_prices.rename(columns = {"date":"Date"})
FIs_prices = FIs_prices.replace(to_replace = 0, method = "ffill")
Price_data_univ=FIs_prices
Price_data_univ = Price_data_univ.set_index("Date") # define Date  as index
# Calculating returns 
Return_data_univ = Price_data_univ.pct_change() #calculate daily returns
Return_data_univ = Return_data_univ.drop(Return_data_univ.index[range(0,1)])

Cov_mat1 = Return_data_univ.cov() # Covariance matrix of the return matrix
Corr_mat1=Return_data_univ.corr() # Correlation matrix of the return matrix
FRM_EM_Adjacency_matrix = FRM_EM_Adjacency_matrix.rename(columns = {"date":""})
FRM_EM_Adjacency_matrix = FRM_EM_Adjacency_matrix.set_index("") # define Date  as index
Corr_mat=FRM_EM_Adjacency_matrix
Cov_mat=FRM_EM_Adjacency_matrix

# In[6]:
# Heatmap and network analysis of corr. matrix

# Plotting Correlation matrix heatmap

plotCorrMatrix(path+"/Adj_matrix_Heatmap_FIs_unsorted",Corr_mat)


# network plot of correlation matrix

plotNetwork(path+"/Corr_Network_FIs_unsorted.png", Corr_mat)

# Sort correlation matrix
dist=correlDist(Corr_mat)
link=sch.linkage(dist,'single')
sortIx=getQuasiDiag(link) 
sortIx=Corr_mat.index[sortIx].tolist() # recover labels 
Corr_sorted=Corr_mat.loc[sortIx,sortIx] # reorder

# Plot sorted correlation matrix
plotCorrMatrix(path+"/Adj_matrix_Heatmap_FIs_sorted",Corr_sorted)

# Plot dendogram of the constituents
#2) Cluster Data
mpl.figure(num=None, figsize=(20, 10), dpi=300, facecolor='w', edgecolor='k')    
dn = sch.dendrogram(link, labels = dist.columns)
mpl.savefig(path+"/Dendrogram_FIs.png", transparent = True, dpi = 300)
mpl.clf();mpl.close() # reset pylab

print(plotNetwork)

# In[7]:
#Function to calculate the HRP portfolio weights

def HRPportf(cov,corr):
    #1) Cluster covariance matrix
    dist=correlDist(corr)
    link=sch.linkage(dist,'single')
    sortIx=getQuasiDiag(link) 
    sortIx=corr.index[sortIx].tolist() # recover labels
    #2) Allocate capital according to HRP
    weights_hrp=getRecBipart(cov,sortIx)
    return weights_hrp


# In[8]:
# Compute the weights for the Markowitz MinVar and the HRP portfolio and the 
# IVP portfolio

w_HRP=np.array([HRPportf(Cov_mat1,Corr_mat1).index,HRPportf(Cov_mat,Corr_mat).round(3)])
w_HRP=pd.DataFrame(np.transpose(w_HRP))
w_HRP.columns = ["Asset","Weights HRP"]

w_MinVar= np.array([min_var_portfolio(Cov_mat1).index,min_var_portfolio(Cov_mat1).round(3)])
w_MinVar=pd.DataFrame(np.transpose(w_MinVar))
w_MinVar.columns = ["Asset","Weights MinVar"]

w_IVP= np.array([Cov_mat1.index, getIVP(Cov_mat).round(3)])
w_IVP=pd.DataFrame(np.transpose(w_IVP))
w_IVP.columns = ["Asset","Weights IVP"]

Weights = pd.merge(w_MinVar,w_IVP,\
                   on="Asset", how = "inner")
Weights = pd.merge(Weights,w_HRP,\
                   on="Asset", how = "inner")

print(Weights.to_latex(index=True)) # Latex table output




```

automatically created on 2021-02-11