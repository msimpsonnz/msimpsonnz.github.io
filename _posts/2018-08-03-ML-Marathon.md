---
layout: post
title: Machine Learning Marathon
---

A quick "run" through on Azure ML Workbench and using it to predict my running ability.

## Running for Starship

My daughter was born with a very rare medical condition, we didn't know this at the time and for the first 4 years were given multiple different diagnosis. We have had and continue to visit Starship children's hospital for numerous procedures, exams and operation and during one of the more serious events she was admitted for over a month in the neurology ward!!

During our visits we have always been amazed at the people and support we have been given during our time there.

Back in 2016 I decided that I wanted to give something back and signed up to the run the Auckland Marathon and raise some money for the Starship Foundation. With a little over 12 weeks to prepare it was always going to be tough but I guess I didn't realise how difficult it was going to be. I think it was so tough that when 2017 came around I was still a bit traumatized and ended up missing out! It was my first long distance run, in fact I have never even done a half marathon until training for the full. I set myself a target of under 5 hours, which I thought was respectable, but I ended up blowing the game plan in the first half and paid for it after 27kms where I just hit the wall and had to hobble to the finish line....5 hours 37 minutes!

Roll forward to now and I have actually found enjoyment out of running and have kept going fairly regularly since, so when I got the email from Starship this year I wanted to get back on the horse. It is just a half marathon this time but I still have just under 14 weeks to prepare. I wasn't sure on a target time so I thought I would use what I have learnt since the first marathon and use machine learning to predict my time.

This is also a shameless plug to donate to Starship and their amazing cause, so please help me help them and donate [here](https://aucklandmarathon2018.everydayhero.com/nz/running-for-starship)

## Data

As part of my preparation for the first marathon I ended up buying a Garmin Forerunner 235, a fitness tracker which had good reviews for running. There is a fair bit of science around training, what not to do and how build up and taper before the big day. So I really wanted to see what I could do with all this data! I thought it was going require way more work, most of these things use proprietary format, but luckily Garmin offer a fairly simple export function on the website. After having to scroll to the end of all my activities I was able to get a export of my running history all the way back to 2016.

[<img src="{{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-1.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-1.png)

## ML Workbench

I know from experience that I was going to need to "wrangle" the data to some degree. I needed to extract the DateTime column and given the "Time" column was a colon separated string!
[ML Workbench](https://docs.microsoft.com/en-us/azure/machine-learning/service/overview-what-is-azure-ml#azure-machine-learning-workbench) to the rescue, I could have used [Pandas](https://pandas.pydata.org/) in [Jupyter Notebooks](http://jupyter.org/) but honestly the data preparation in Workbench is amazing. It has this awesome "Derive column from example" feature where you give it a source column and some examples of the data you want extracted and it uses ML to go figure that out! I know, ML to do ML, just like inception.

### Importing CSV
[<img src="{{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-2.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-2.png)


### Data types
[<img src="{{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-3.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-3.png)


### Data preparation workflow
[<img src="{{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-4.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-4.png)

## Launch Jupyter

Ok so now it was time for Jupyter, these Notebooks are great the offer a simple environment to create a live document for a variety of coding languages, great for documentation and comments, charts and live code. I really like them for Python as I can run a single command without having to use a shell (REPL) or saving to a script and executing over and over.

I used Pandas which is a great Python library for data, I extracted the "Features" from my transformed dataset
```features = df[["Year","Month","Day","Hour","Distance","Elev Gain", "Elev Loss"]]```

I had a string column in the dataset for location of the activity and needed to convert this to something the model could actually use, so I used the Pandas "Dummies" features which basically gets all the unique strings in the column and pivots the data so that I get a new column for each string and a "1" or "0" as the value, this is often referred to as ["One Hot" encoding](https://machinelearningmastery.com/how-to-one-hot-encode-sequence-data-in-python/).

I had used Workbench to extract "Hours", "Minutes" and "Seconds" from my actual running time, I then needed to combine these to make my label for the model. I decided just to convert these into seconds and add them all together.
```
labelsraw = df[["Act_Hour", "Act_Min", "Act_Sec"]]
labels = labelsraw["Act_Hour"]*60*60 + labelsraw["Act_Min"]*60 + labelsraw["Act_Sec"]
```

So now I have my features and label, I need to split the dataset and train a model. We split the data for training so we end up with a "blind" trial. I build the model using the "training" set and then use the "test" data which the model has never seen before to see how good it will be.

```
from sklearn.model_selection import train_test_split
X_train, x_test, y_train, y_test = train_test_split(data, labels, test_size=0.7, random_state=7)

from sklearn import linear_model
model = linear_model.LinearRegression()
model.fit(X_train, y_train)

model.score(x_test, y_test)
```

So what did we get..... `0.9612048122276895`

### Notebook
[<img src="{{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-5.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-5.png)

## IRL

OK! 96% accurate, not bad but it's a limited dataset and only has a few entries of long distance (10k plus).

Lets try this in real life, I'm off for a run stay right there......

Cool, so I managed `4.72km in 29.44`

The computer says...

```
predictRun = np.array([2018,7,27,12,4.72,40,39,0,0,0,0,0,0,1,0,0,0,0]).reshape(1,-1)
rawResult = model.predict(predictRun)
resultRun = str(datetime.timedelta(seconds=rawResult[0]))
```

[<img src="{{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-6.png" style="width: 600px;"/>]({{ site.baseurl }}/images/2018-08-03-ML-Marathon/ml-6.png)

So `29.44 actual` vs `29.53 predicted` not bad less than 10 second error margin.

But what about the half marathon...

`1:40:04` WHAT!!!! I need to do some more training! 

Like I said this data set doesn't have enough long distance activities so I think it is being way over bias on this! I'll revisit this along the way with some more data.