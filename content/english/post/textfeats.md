---
author: Makayla Whitney
date: "2022-10-03"
description: model development process
tags:
- markdown
- text
series:
- series-setup
tags:
- readability
- textfeatures
- postagging
title: Developing Text Features
---
# Dataset

The readability model featured in this series was developed in EDUC 654: Machine Learning taught by Dr. Cengiz Zopluoglu at the University of Oregon. Text features as well as the model were created with text data from the [CommonLit Readability Kaggle Competition](https://www.kaggle.com/c/commonlitreadabilityprize/ "CommonLit Readability Kaggle Competition"). This competition urged users to build models that rate the complexity of reading passages found in 3rd - 12th grade classrooms. Our class took the data provided by this competition and developed a model that would allow us to predict the readability of any text supplied. In order to do this, we started with developing text features to feed into the model. 

# Pre-Processing Text Data

In order to predict the readability of a plain text excerpt, we need to start with deriving accompanying features. The following code chunks will display the process of building a function to derive text functions based on the above mentioned Kaggle competition. 
```
generate_feats <- function(my.model,new.text){

```
## Tokenization
To derive features we will mainly be using packages `quanteda` and `quanteda.textmodels`. The first thing to begin with is tokenization of the plain text. This separates out all words, forms of punctuation, symbols, and numbers present in the plain text. 
> For example, the sentence "The brown dog barked and ran after the ball." would be separated into "the", "brown", "dog", "barked", "and", "ran", "after", "the", "ball", ".".

Once the plain text has been tokenized, we created a document-feature matrix that contains information about the frequency of each token found in the text. 
```
tokenized <- tokens(new.text,
                    remove_punct = TRUE,
                    remove_numbers = TRUE,
                    remove_symbols = TRUE,
                    remove_separators = TRUE)
  
dm <- dfm(tokenized)
```
### Text Statistics
The next step is to develop some basic statistics about the plain text. We do this via the `textstat_summary` function. This allows us to derive the following total amounts of information from a block of text: number of characters, sentences, tokens, unique tokens, numbers, punctuation marks, symbols, tags, and emojis. 
```
text_sm <- textstat_summary(dm)
text_sm$sents <- nsentence(new.text)
text_sm$chars <- nchar(new.text)
```
### Word Length
We also derive the distribution of word lengths in the plain text. The number of words with lengths between 1-20 can be used as predictive features within the readability model. These word length statistics are made up of information regarding minimum and maximum word length, average word length, and variability of word length. It is important for teacher and literacy practitioners to understand the variability of word length when choosing reading materials for their individual students; word length contributes to overall comprehension of a passage. 

```
wl <- nchar(tokenized[[1]])
  
wl.tab <- table(wl)
  
wl.features <- data.frame(matrix(0,nrow=1,nco=30))
colnames(wl.features) <- paste0('wl.',1:30)
  
ind <- colnames(wl.features)%in%paste0('wl.',names(wl.tab))
  
wl.features[,ind] <- wl.tab
  
wl.features$mean.wl  <-   mean(wl)
wl.features$sd.wl    <-   sd(wl)
wl.features$min.wl   <-   min(wl)
wl.features$max.wl   <-   max(wl)
```
### Text Entropy
Within the `quanteda.textstats` package there is a function called `textstat_entropy()` that is used to calculate a numerical measure of text entropy. Entropy of language is a statistical parameter that accounts for the average amount of information produced for each letter of a text. The concept of information entropy was first introduced in 1948 by Claude Shannon. After extensive trials in which participants identified and guessed words within a string of words, it was found that there is neither high or low entropy, but instead an optimal range (1). For English text, this optimal range is between 1.0 and 1.5 bits per letter. The more text that falls within this range, the more comprehensible a passage is. 
```
t.ent <- textstat_entropy(dm)
n     <-  sum(featfreq(dm))
p     <- rep(1/n,n)
m.ent <- -sum(p*log(p,base=2))
  
ent <- t.ent$entropy/m.ent
```
### Lexical Diversity
The `textstat_lexdiv()` function uses the number of unique tokens (V) and the total number of tokens (N) to calculate the lexical diversity of a text using 13 indices. More information regarding the formulas and descriptions of these indices can be found on the help page (?textstat_lexdiv). 
```
text_lexdiv <- textstat_lexdiv(tokenized,
                                remove_numbers = TRUE,
                                remove_punct   = TRUE,
                                remove_symbols = TRUE,
                                measure        = 'all')
```
### Measures of Readability
For decades, there have been a variety of measures of readability in existence and use. The `textstat_readability()` function creates 46 different indices as functions of these measures. Some of these include the number of words, characters, sentences, and syllables, words matching and difficult words not matching the [Dale-Chall List of 3000 familiar words](https://www.readabilityformulas.com/articles/dale-chall-readability-word-list.php "Dale-Chall List of 3000 familiar words"), and average sentence and word lengths. 
```
text_readability <- textstat_readability(new.text,measure='all')
```
## Lemmatization
The next step in deriving text features involves lemmatization, or the process of splitting each word into its meaningful base or root form (otherwise known as the 'lemma'). This requires the `udpipe` package and downloading a pre-made language model. The `udpipe` package has models for 65 different languages; for this project, we used the model for English. The following code chunk displays how to install and download the necessary tools.
```
require(udpipe)
udpipe_download_model(language = "english")
ud_eng <- udpipe_load_model(here('english-ewt-ud-2.5-191206.udpipe'))
```
### Part of Speech Tags (POS)
A part of speech tag is added to each of the lemmas disseminated from the previous code. These values can be quantified to give information about a block of text. They can also be used as a feature in determining overall readability of a passage. 
The part of speech tags come from a list of [Universal POS Tags](https://universaldependencies.org/u/pos/index.html "Universal POS Tags"). More information regarding their abbreviations can be found on [Oxford English part-of-speech Tagset](https://www.sketchengine.eu/oxford-english-corpus-tagset/ "Oxford English part-of-speech Tagset").
```
annotated <- udpipe_annotate(ud_eng, x = new.text)
annotated <- as.data.frame(annotated)
annotated <- cbind_morphological(annotated)
  
pos_tags <- c(table(annotated$upos),table(annotated$xpos))
    
```
### Syntactic Relations and Morphological Features
Another text feature developed is syntactic relations. There are 37 tags for [universal syntactic relations](https://universaldependencies.org/u/dep/index.html "universal syntactic relations"). There will also be features of the text not tagged with a part of speech. These include grammatical and lexical aspects of the plain text. These are labeled with their appropriate morphological tag. 
```
# Syntactic relations
  
dep_rel <- table(annotated$dep_rel)
  
# Morphological features
  
feat_names <- c('morph_abbr',
                'morph_animacy',
                'morph_aspect',
                'morph_case',
                'morph_clusivity',
                'morph_definite',
                'morph_degree',
                'morph_evident',
                'morph_foreign',
                'morph_gender',
                'morph_mood',
                'morph_nounclass',
                'morph_number',
                'morph_numtype',
                'morph_person',
                'morph_polarity',
                'morph_polite',
                'morph_poss',
                'morph_prontype',
                'morph_reflex',
                'morph_tense',
                'morph_typo',
                'morph_verbform',
                'morph_voice')

feat_vec <- c()
      
for(j in 1:length(feat_names)){
        
    if(feat_names[j]%in%colnames(annotated)){
          morph_tmp   <- table(annotated[,feat_names[j]])
          names_tmp   <- paste0(feat_names[j],'_',names(morph_tmp))
          morph_tmp   <- as.vector(morph_tmp)
          names(morph_tmp) <- names_tmp
          feat_vec  <- c(feat_vec,morph_tmp)
        }
      }
```
## Natural Language Processing (NLP)
Natural Language Processing (NLP) is the process of breaking text into parts (i.e. tokens, lemmas, words, POS tags) and converting information gathered via a neural network model into a numerical vector that holds meaningful data representative of the text. For more information regarding NLP, please refer to Dan Jurafsky and James H. Martin's book, **Speech and Language Processing** We use NLP when extracting text features to calculate sentence embeddings. NLP requires that we use packages from both Python and CRAN to complete the process here. NLP models provide meaningful contextual and numerical representations of words or sentences. These representations can be used as feature for predictive models to assist in obtaining a certain outcome - such as readability. Below is a code chunk modeling how to download the necessary packages from Python. 
```
require(reticulate)

# List the available Python environments

virtualenv_list()

# Import the modules

reticulate::import('torch')
reticulate::import('numpy')
reticulate::import('transformers')
reticulate::import('nltk')
reticulate::import('tokenizers')

# Load the text package

require(text)

```
### Sentence Embeddings
An entire sentence can be represented using a numerical vector. `textEmbed` allows us to return a matrix of text embeddings for each word in the plan text as well as a vector of embeddings for the whole sentence. This matrix is comprised of several dimensions in which the sentence is represented wothin. 
```
embeds <- textEmbed(x = new.text,
                    model = 'roberta-base',
                    layers = 12,
                    context_aggregation_layers = 'concatenate')

```
## Final Steps
The following code chunk displays the final steps in creating text features to contribute to our readability model. 
```
# combine them all into one vector and store in the list object
      
input <- cbind(text_sm[2:length(text_sm)],
                          wl.features,
                          as.data.frame(ent),
                          text_lexdiv[,2:ncol(text_lexdiv)],
                          text_readability[,2:ncol(text_readability)],
                          t(as.data.frame(pos_tags)),
                          t(as.data.frame(c(dep_rel))),
                          t(as.data.frame(feat_vec)),
                          as.data.frame(embeds$x))

# feature names from the model
  
my_feats <- my.model$recipe$var_info$variable

# Find the features missing from the new text
      
missing_feats <- ! my_feats %in% colnames(input)
      
# Add the missing features (with assigned values of zeros)
        
temp           <- data.frame(matrix(0,1,sum(missing_feats)))
colnames(temp) <- my_feats[missing_feats]
   
input <- cbind(input,temp)
      
return(list(input=input))
}
```

Please move on to the next post in this series **Readability Model** to continue learning how to build our predictive model. For more information regarding creation of text features, refer to Dr. Zopluoglu's lecture [Data Pre-Processing II (Text Data)](https://ml-21.netlify.app/notes/lecture-2b.html "Data Pre-Processing II (Text Data)"). 


Resources
1. Moradi, H., Grzymala-Busse, J. W., & Roberts, J. A. (1998). Entropy of English text: Experiments with humans and a machine learning system based on rough sets. Information Sciences, An International Journal, 104, 31-47.