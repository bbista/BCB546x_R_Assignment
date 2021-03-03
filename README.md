# BCB546x_R_Assignment
---
title: "Code_Workflow"
author: "Basanta Bista"
date: "October 12, 2018"
output:
  pdf_document: default
  html_document: default
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Part I:  Unix assignment in R

Loading the tidyverse package:
```{r}
if (!require("tidyverse")) install.packages("tidyverse")
library(tidyverse)
```

## Downloading files

```{r} 
# Downloading files directly from the github repository 
snp<-read_tsv("https://raw.githubusercontent.com/EEOB-BioData/BCB546X-Fall2018/master/assignments/UNIX_Assignment/snp_position.txt")
genotypes<-read_tsv("https://raw.githubusercontent.com/EEOB-BioData/BCB546X-Fall2018/master/assignments/UNIX_Assignment/fang_et_al_genotypes.txt")
```
## Data Inspection

First, we need to inspect the data

```{r}
# Load data:
str(snp)
str(genotypes)
unique(genotypes$Group)
unique(snp$Chromosome)
```



### Data Processing

Rearranging the SNP file so that the Chromosome column is second followed by position, followed by transposition of the SNP file

```{r}
# Rearranging the snp file
snp<-snp[c(1,3,4,2,5:15)]
#Subsetting the genotypes dataframe for the maize and teosinte group seperately
maize.genotype<-genotypes[genotypes$Group=="ZMMIL"|genotypes$Group=="ZMMLR"|genotypes$Group=="ZMMMR",]
teosinte.genotype<-genotypes[genotypes$Group=="ZMPJA"|genotypes$Group=="ZMPIL"|genotypes$Group=="ZMPBA",]
```

### Transposing the subset of genotypes to merge with the SNP data frame
```{r}
#Transposing the dataset and converting the result of a t() function  i.e a matrix into dataframe
maize.genotype<-as.data.frame(t(maize.genotype), stringsAsFactors = F)
teosinte.genotype<-as.data.frame(t(teosinte.genotype), stringsAsFactors = F)
#Column names have been transposed to the row name, we create another row from the row names called SNP_ID  The first row is then converted to the row names and remove the fisrt three columns. 
SNP_ID <- rownames(maize.genotype)
rownames(maize.genotype) <- NULL
maize.genotype <- cbind(SNP_ID,maize.genotype,stringsAsFactors = FALSE)
names(maize.genotype)<-  c("SNP_ID",maize.genotype[1,-1])
maize.genotype <- maize.genotype[-c(1,2,3), ]
SNP_ID <- rownames(teosinte.genotype)
rownames(teosinte.genotype) <- NULL
teosinte.genotype <- cbind(SNP_ID,teosinte.genotype,stringsAsFactors = FALSE)
names(teosinte.genotype)<-  c("SNP_ID",teosinte.genotype[1,-1])
teosinte.genotype <- teosinte.genotype[-c(1,2,3), ]
```

Now, the transposed genotype file is merged with the snp info file:
```{r}
maize.merge<-merge(snp, maize.genotype, by="SNP_ID")
teosinte.merge<-merge(snp, teosinte.genotype, by="SNP_ID")
```
We now subset the merged dataset based on Chromosome number, sort it based on position and export it in the form of csv files

```{r error=TRUE}
#THis code first subsets the merged data based on Chromosome with numerical valies i.e 1-10 then groups it by the Chromosome number, folloed by sorting by the Position in ascending order and pipes them into a write.csv function that will export the files as csvs with names based on the Chromosome number, order and group (Maize/Teosinte)
#The as.numeric function will show an error message when it encounters a character strinf i.e "unknow/multiple" value which will be replace by NA
#The do function will execute a function on the piped group wise dataset, in this case export them as csv. THe do() function will show an error messsage when a datafram is not the output of the do() function, neverthless, the code still works and individual files for each chromosome are still created. 
maize.merge %>% filter(Chromosome %in% c(1:10))%>%  group_by(Chromosome) %>% arrange(as.numeric(Position)) %>%   do( write.csv( .,paste0("Maize_Chr_",unique(.$Chromosome),"_ascending.csv"), row.names = F))
teosinte.merge%>% filter(Chromosome %in% c(1:10)) %>%  group_by(Chromosome) %>% arrange(as.numeric(Position)) %>%   do( write.csv( .,paste0("Teosinte_Chr_",unique(.$Chromosome),"_ascending.csv"), row.names = F))
```
For the second part of the question, we will now replace "?/?" with "-/-" and then export them in descending order of positions 
```{r error=TRUE}
#replace all ? with -
maize <- data.frame(lapply(maize.merge, gsub, pattern = "?", replacement = "-", fixed = TRUE))
teosinte <- data.frame(lapply(teosinte.merge, gsub, pattern = "?", replacement = "-", fixed = TRUE))
 
#exporting files
maize%>% filter(Chromosome %in% c(1:10)) %>%  group_by(Chromosome) %>% arrange(desc(as.numeric(Position))) %>%   do( write.csv( .,paste0("Maize_Chr_",unique(.$Chromosome),"_descending.csv"), row.names = F))
teosinte%>% filter(Chromosome %in% c(1:10)) %>%  group_by(Chromosome) %>% arrange(desc(as.numeric(Position))) %>%   do( write.csv( .,paste0("Teosinte_Chr_",unique(.$Chromosome),"_descending.csv"), row.names = F))
```



# Part II

Data Visualization.

### SNPs per chromosome
Visualization of number of possible polymorphism position in each chromosome 


```{r}
library(ggplot2)
#Viusalizing polymorphism position in each chromosome
ggplot(data = snp[!is.na(as.numeric(snp$Chromosome)),]) +   geom_bar(mapping = aes(x = as.numeric(Chromosome))) + scale_x_discrete(limit=c(1:10))+ labs(x = "Chromosome"
, y="No. of polymorphism position")
```

```{R}
#tidying up the data using melt
library(reshape2)
a<-melt(genotypes, id.vars=c("Sample_ID", "JG_OTU", "Group"), variable.name = "SNP_ID", 
  value.name = "Base")
a<-merge.data.frame(a, snp, by="SNP_ID")
ggplot(data = a[!is.na(as.numeric(snp$Chromosome)),]) +   geom_bar(mapping = aes(x = as.numeric(Chromosome), fill=Group)) + scale_x_discrete(limit=c(1:10))+ labs(x = "Chromosome"
, y="No. of polymorphism position")
```
```{R}
#Graphs that represent polymorphism contributed by each group
graph <- a %>% 
  mutate(Chromosome=as.numeric(Chromosome)) %>% #Mutate Chromosone number into numeric
  select(Group, SNP_ID, Chromosome, Base) %>%  #Selcet required fields
  filter(Chromosome %in% c(1:10)) %>%  #Filter chromosome 1 to 10
  filter(Base!="?/?")%>%   #remove all unknown bases
  group_by(Group,SNP_ID) %>%  #group by Group and SNP_ID
  filter(length(unique(Base))>1) %>%   #remove all SNPS in a group that has only one base i.e no polymorphism in the group
  select(Group, SNP_ID, Chromosome) #select requred fields
graph<-graph[!duplicated(graph),] #remove duplicated records
ggplot(data=graph) +
  geom_bar(mapping=aes(x=Chromosome, fill=Group)) + 
  scale_x_discrete(limit=c(1:10), label=c(1:10))
  #plotting the filtered data based on polymorphism contributed by each group
```

Melting the genotypes data 

```{r}
#creating a new column based on homozygousity of the polymorphism
a$homozygous<-TRUE
a$homozygous[a$Base=="?/?"]<-NA
a$homozygous[substr(a$Base,1,1)!=substr(a$Base,3,3)]<-FALSE
# Graph the SNPs by Group, filling by Homozygousity:
graphs<-a
ggplot(data=a) +
  geom_bar(mapping=aes(x=Group, fill=homozygous), position="fill") 
  
```

Creating multiple graphs based on polymorphism is each group in every chromosome
```{r, fig.width=10,fig.height=11}
b<-a[!is.na(as.numeric(a$Chromosome)),]
b$Chromosome<-as.numeric(b$Chromosome)
b<-b%>% select(SNP_ID, Group, Base, Position, Chromosome)%>% filter(Base!="?/?")
b<-b[!duplicated(b),]
b<-b[!is.na(as.numeric(b$Position)),]
g<-ggplot(data = b) + geom_point(mapping=aes(x=as.numeric(Position), y=Group, color=Base)) +labs(y = "Group" , x="Chromosome position")
g + facet_wrap(~ Chromosome,ncol=2)
```
