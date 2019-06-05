---
layout: post
title: Discord Social Graph
subtitle: Machine-Learning woo to terrorize your local Discord Server
comments: true
published: true
---

## Please be patient for link to load. I'm using a Heroku Hobby Dyno, which shuts down and has to restart.

https://infinite-sands-83078.herokuapp.com/

Please be patient, I'm using a free-tier Heroku account, so it's very slow to initially respond and to load. You may have to wait around a minute.

### Intro

DiscordSocialGraph is my first original Machine Learning project. Originally, I set out to create a Discord bot that would predict the next user to join the server voice-chat. However, that revealed a more interesting prospect: who is the most popular person on our server? Another way to ask this would be: which user has the most "draw" to get users to join the server?

I started by creating an [addon](https://github.com/samclane/Snake-Cogs/blob/master/member_logger/member_logger.py) to the existing Discord Bot framework that I use ([RedBot](https://github.com/Cog-Creators/Red-DiscordBot)). I wanted to collect as little information as possible (to start simple), so all this module does is log 2 things:


1. When a user joins a voice channel, it logs the user IDs of the members already in the channel. 

2. When a user mentions another using `@`

Here's a mockup of what that file looks like:
```
timestamp,member,present
1539291278,user3,"['user8', 'user1', 'user0', 'user7', 'user4', 'user6']"
1539291514,user1,"['user5', 'user0', 'user4', 'user2', 'user7', 'user8', 'user9', 'user6']"
1539292425,user0,"['user7', 'user5', 'user3', 'user9', 'user4']"
1539293267,user3,"['user4', 'user0', 'user2', 'user7', 'user6']"
1539293442,user9,['user8']
1539293609,user3,"['user6', 'user0', 'user7', 'user4']"
1539293634,user3,"['user0', 'user6', 'user5', 'user8']"
1539293848,user9,"['user6', 'user4', 'user0', 'user7']"
1539294307,user9,"['user6', 'user1', 'user0']"
1539294408,user6,"['user1', 'user4', 'user0', 'user7', 'user5', 'user2']"
1539295361,user6,"['user0', 'user1', 'user4', 'user3', 'user9', 'user2']"
```
<sup>Note: User IDs have been changed to userX for readability</sup>

The bot collects this information and uploads it to a remote PostgresSQL server requisitioned by Heroku. 

The list of all users with interactions on the server is kept as a one-hot vector. The user is treated as the label. The classifier has to use a probabilistic OneVsAll approach, giving a probability distribution over the entire user-base instead of just the top answer. Using this method across the entire userbase generates a distribution of the one-way probability of a given User X interacting with another User Y. This will generate a __graph__:

![](https://i.imgur.com/tVC6XqZ.png)

<sup>Note: This graph uses the [Fruchterman Reingold layout algorithm](https://github.com/gephi/gephi/wiki/Fruchterman-Reingold) which tries to display the Graph in a spatially meaningful way. Unfortunately that doesn't always work.</sup>

The "popularity" or "draw" of the user is the sum of the weights of all the in-degree weights. Currently, this correlates pretty heavily with the number of instances that user appears in the dataset, but not exactly, meaning that some special relationships are being discovered. For example, it's noticed that `watersnake_test`, my test account, is almost exclusively joining when I'm already in the server (I only use it when I need to simulate another user besides myself). However, the algorithm can sometimes overfit, drawing strong bonds between my test-account and other accounts I'm actually friends with. It's interesting to try and see the model try and discover who's friends with who. 

Three different models were used in the development of this project:

1. Naive Bayes (`sklearn.naive_bayes.GaussianNB`)
2. Support Vector Machine (`sklearn.svm.SVC`)
3. Multilayer Perceptron (`sklearn.neural_network.MLPClassifier`)

Currently, model #3, the `MLPClassifier`, gives the most accurate results. The accuracy of the model is quantified by the area under the Receiver Operating Characteristic curve. 

![](https://i.imgur.com/eibcGPe.png)

Basically, it describes the correct guesses (True Positive rate) against the wrong guesses (False Positive Rate) as the threshold for classification is narrowed. 

### The Webapp

Since this application uses information collected from other people, it was suggested to put it online for all to see. The app is hosted on a free Heroku account, with a Hobby Dyno. The heavy ML lifting is done with a RedisQueue background job, started as soon as the first HTTP request comes through. It then displays a loading message while the webapp boots up. The model is retrained from scratch each time it's restarted, as it doesn't take *that* long and gives the most recent, accurate result as obtained from the database.

### Results

From what I've gathered, and from what I can interpret from the metrics returned by the test data set, the classifier is working better than random guessing, by roughly 25%. The way that popularity is calculated could use some work, as it really does heavily on the non-uniform distribution of user participation for the majority of its "Accuracy". Also, some users who have rarely visited the server have really strong bonds with several other users, as a certain user could always be in the server, leading to a 100% bond strength between the outlier and the central user(s). 