# Predicting User-News Interactions

Amanda Cheney  

March 1, 2021 

### Goal 

Construct a deep learning network using NLP techniques that is able to predict whether a user will click on an unseen news article. The model should take as input user-news items and output a binary variable (1/0).

### Data 

We are provided a dataset of nearly 11 million user-news interactions between 50,000 unique users and 51,282 unique articles. Each of these nearly 11 million user-news interactions, however, is not necessarily unique. EDA reveals that a given user may interact with a given news article more than once. In some cases the user may see an article once and click on it, but see the same article a second  time and not click on it (or vice-versa), or in the case of a few users, click on the same article over 300 times. For the purposes of this analysis, however, whether a user saw an article 300 times and clicked on it all 300 times or saw it 300 times and never clicked on it, all we are interested in is whether or not there was at least one click. Therefore, after grouping the data by `user_id` and `item` and summing the total number of clicks, I binarized a given user's total number of clicks into a simple 0 or 1 to serve as the target variable, `target` . This left me with a dataset of 5,894,984 unique user-item interaction combinations, nearly half the size of the original dataset, which I then merged with the news dataset for my final DataFrame. 

EDA using Pandas revealed a notable class imbalance on the `target` variable. Only 19.5% of unique user-item interaction combinations result in a click, while over 80.5%  never result in a click. 

### Approach 

1. <u>Training/Testing Methodology</u>: Before designing my model, I divided my data into train and test sets, while also ensuring that both sets reflected the roughly 80/20 class imbalance in the data. Later when we turn to building the classification model we will further divide the training data into train and validation sets, which is a function built into fast.ai's data block API - more on that below.
2. <u>Preprocessing</u>: FastAI handles both tokenization and numericalization "under the hood" with one simple call using `TextList` which converts the text data of article titles and abstracts into a list of tokens and then convert those tokens into a numeric representation, or `databunch` which is then fed to the actual language model itself. 
3. <u>Build and train deep learning network</u>: I used FastAI, a relatively new ML library built on top of PyTorch, known for its simple training approach as well as for providing state-of-the-art results in standard deep learning domains, including NLP, by enabling users to use utilize both transfer learning and self-supervised learning to fine-tune pertained language models. 
4. <u>Evaluate model performance</u>: The 80/20 class imbalance has important implications for how we evaluate model performance. Most importantly, we should avoid relying on model accuracy alone, since accuracy applies a "naive" 50% threshold to decide between classes, and that is not applicable here. In the "worst case scenario", our model could simply predict the majority class all the time (i.e. no click) and have fairly high accuracy, but such a model wouldn't offer much insight into what we are actually interested in, which is ultimately those articles that do get clicks. In a business application, it would likely be the case  that **precision is the most important evaluation metric**. This is because precision punishes false positives while not caring about the rate of false negatives. Here, a false positive would be inaccurately labelling an article that was not clicked as clicked, which in the context of a news recommender would mean showing a user an article they do not want to see and are not going to click on, which could have negative effects on user engagement and customer churn. That said, this same type of model applied to a different context, however, such as one which aims at educating users and ensuring that they have access to as many of the relevant news items as possible, might instead choose to prioritize recall performance rather than precision. 

### Model Architecture 

Building my deep learning network in FastAI involved two key stages. First, creating a "language model" and second, creating the actual classification model. 

**Language Model:** The architecture of my language model is built using `AWD_LSTM`. AWD-LSTM is a pre-trained language model trained on Wikipedia text data (Wikitext-103). While theoretically, we could directly use this pre-trained language model to build our news classifier, the language used in news headlines and abstracts is somewhat different from the language of Wikipedia. FastAI allows us to fine-tune this pre-trained language model using our news abstract and title data and then build our classifier on top of that to create our own language model. 

The language model we are building here, is being trained to guess the next word starting from a sequence of words as input. It has a recurrent structure and a hidden state that is updated each time it sees a new word. While simple RNN's suffer from the exploding gradients, using LSTM (Long Short Term Memory) helps avoid this problem as it has a unique way of computing the hidden state and is capable of learning long-term dependencies. The AWD aspect of the AWD-LSTM architecture (which stands for **A**SGD **W**eight **D**ropped  (ASGB stands for **A**veraged **S**tochastic **G**radient **D**escent)) takes a regular LSTM with several layers and adds on additional regularization using various types of dropouts which discard partial information of the weight matriculates between the hidden states to avoid overfitting. 

We go through several rounds of training, unfreezing the model and retraining with different learning rates,  fine tuning to select the best model. In this stage, where we are focused on building the language model and not classification,  we use accuracy to evaluate model performance. One the initial round of training is completed I continued to fine tune the model for 4 epochs until I reached an accuracy of 90.9%. That means that in 90% of cases, my model is able to correctly predict the next word. 

Once this is done, we save all of our model except the final layer that converts activations to probabilities of picking each token in our vocabulary. The model not including the final layer is called the *encoder*. 

**Classification Model:** Once the language model is sufficiently fine tuned, I then utilized it to fine tune a classifier, which unlike the language model, uses the target feature labels. 

To make our classifier, we first need to create a new `databunch` -- following essentially the same steps as we did to create the language model databunch. The only difference is that here we take 20% of the training data to serve as a validation set and keep our vocabulary the same as that of the language model databunch.  At this stage, we also add our test data. 

To build the actual classifier, we use the *encoder* from our language model as well as the same `AWD-LSTM` architecture using FastAI's `text_classifier_learner`. Here we also have the opportunity to specify evaluation metrics. Beyond accuracy, I'm also interested in recall, precision and FBeta. I've set FBeta = 0.5 because, while both precision and recall are important, I put more emphasis on precision as discussed previously. 

Once again, we go through several rounds of training, unfreezing the model and retraining with different learning rates,  fine tuning to select the best model. With the 3 final epochs producing the following levels of model performance, I end training. 

| epoch | train_loss | valid_loss | accuracy | recall   | precision | f_beta   | time    |
| ----- | ---------- | ---------- | -------- | -------- | --------- | -------- | ------- |
| 0     | 0.264991   | 0.190445   | 0.942365 | 0.743460 | 0.950534  | 0.900378 | 1:15:48 |
| 1     | 0.222628   | 0.187756   | 0.943670 | 0.741195 | 0.961424  | 0.907496 | 1:19:02 |
| 2     | 0.191943   | 0.184952   | 0.944512 | 0.742678 | 0.965033  | 0.910512 | 1:13:43 |

**Model Performance **

With a final precision score of 96.5% our model is performing very well. Looking at the final model's confusion matrix, however, we see that much of this performance is still being driven by the majority ('no click') class. 26% of 'clicked' articles are still being mis-labelled as not clicked. While we have not prioritized recall as an evaluation metric, ideally in future work, we would be able to arrive at even higher scores than what we have produced here. One key element would be to take additional steps to ensure proportional class representation within FastAI's train-validation split within the classifier. This functionality is not built into the FastAI, however, with additional time, it would likely be possible to work around this using sklearn. 



![conf_matrix](/Users/AmandaCheney/Downloads/conf_matrix.png)



### Model Deployment & Monitoring 

While this exercise has focused on the development of a model to predict user-news interactions, in practice, development is only just the beginning a ML model's lifecycle. Once created, the next step is to put the model into production which includes not only deploying the model but also monitoring it. Some of these challenges include: computing power, portability, scalability, and language change. FastAI excels in terms of ease of use but, compared to say TensorFlow+Keras, is less mature and does not have the same native systems for deployment or levels of mobile or high scalability server capabilities. Deploying a fast.ai model has historically involved setting up and self-maintaining a customized inference solution (e.g., with Flask, which requires additional maintenance of security, load balancing, services orchestration, etc and is time consuming. However, newer tools like [TorchServe and Amazon SageMaker](https://aws.amazon.com/blogs/opensource/deploy-fast-ai-trained-pytorch-model-in-torchserve-and-host-in-amazon-sagemaker-inference-endpoint/) offer promising solutions for deploying FastAI/PyTorch models at scale and the opportunity to make real-time inferences via a REST API. 

Another key question to consider is how frequently do predictions need to be generated? Will predictions need to be generated for a single instance on demand - such as recommendations offered to users each time they login to the website or app? Or will they be generated in batches such as a daily/weekly/monthly marketing email where recommendations can be computed for all users at the same time and cached? Each of these scenarios have very different latency requirements. Batch predictions allow the DS team to take advantage of scalable computing resources by generating recommendations on some recurring schedule, but these predictions/recommendations are not available for real time purposes. Single instance predictions, by contrast, allow users to request and receive predictions/recommendations as soon as they are needed in real time, but have higher latency requirements and can be more vulnerable to model drift. 

On the subject of model drift, this will be a key challenge in monitoring. Drift happens when the statistical properties of your data cases the quality of your predictions to change over time. In this case of our news recommender, as the content of current events change overtime, our model may not be as successful at predicting whether or not a user will click on a given news article. Successful monitoring of drift depends on how quickly you have access to the ground truth. Luckily, we do not need to wait weeks into the future to know if users do or do not click on an article, so monitoring performance with standard measure like accuracy, precision, recall and FBeta should be sufficient. Were we not to have timely labels, however, we would need to look at other methods. 

## Business Use Case - Final Thoughts 

While the goal of this particular challenge was to build a deep learning model specifically, I will conclude by considering the question of whether or not a neural network is actually the best type of model for this particular application. The answer here will depend on whether the exact business use case  aims to optimize model interpretability or performance. If performance is the goal, then a neural network may indeed be the optimal choice. However, neutral networks are among the least interpretable models, as exactly what's going on under the hood to render this performance is within a black box that is not possible to understand. By contrast, if interpretability is the goal, then modeling techniques other than a neural network would be a more optimal choice.

Two alternative approaches come to mind, both combining unsupervised and supervised learning. One would be to use topic modeling. Topic modeling assumes that each document in a corpus is a combination of a fixed number of topics. After generating the topics, each document then has a distribution of topics (say you had 3 topics, some might be 100% topic 1, others might be 20%/70%/10% topic 1/2/3 and so on ). These topic distributions could then be used at feature vectors in a simple, highly interpretable, supervised classification model like logistic regression - perhaps also in combination with other available item features, such as length of article/title/abstract, article sentiment, in addition to other variables that were not available within the confines of this challenge such as publication timing, article tags, author and so on.  

Another, similar approach would be to create document embeddings--potentially employing transfer learning--and generating clusters of different types of news articles from these embeddings, with each article assigned to a different cluster which we could then in put into a logistic regression model along the lines described above. These approaches, while perhaps less sophisticated than a neural network would also lend well to a recommender system where users can be recommended articles based on their levels of interest in the various topics or clusters. 

In the context of a news recommender system where the firm deploying the recommender system is also responsible for creating content, understanding the features of articles that get clicks vs those that do not could be critically important to the larger business goals of the company. However, this may not be the case of the firm was strictly in the business of recommending articles not creating them themselves. 

An final factor to consider is that in the context of a news recommender, the range of possible topics or kinds of clusters lend fairly well to interpretability by a data scientist with moderate domain knowledge, but not necessarily high levels of subject matter expertise. This is unlikely to be the case for content recommender system in a context of an online platform like Inspire, which hosts over 140 communities for different health conditions. In this context, it would be all but impossible for a data scientist to wield enough domain knowledge to successfully interpret topics from a topic modeling approach or the groupings that would emerge from clustering document embeddings. Such an endeavor would require intensive collaboration with a larger team, which, while not inconceivable, would entail significantly more time and labor than that which would go into building out an effective neural network. 

