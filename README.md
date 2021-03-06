# A Machine Learns to Predict the Stability of Circumbinary Planets

This is a tutorial for using a deep neural network (DNN) to predict the orbital stability of circumbinary planets, as well as for training and validating your own DNN. Our DNN was trained on one million (well, not really one million; see "Graduating from slices") simulations run on [REBOUND](http://rebound.readthedocs.io/en/latest/index.html), a numerical integrator by Hanno Rein et al. You can use our code to generate stability predictions for the circumbinary planetary system of your choice. As a proof of concept, this model is simplified - one can imagine introducing additional parameters, such as orbit inclination. 

To run predictions on the stability of a circumbinary system given binary eccentricity, binary mass ratio (µ), and binary and planet semi-major axes, simply run

```
python tatooine.py -a ____ -e ____ -m ____
```

where the quantity following the -a flag is the ratio of the planet's semi-major axis to the binary semi-major axis; the quantity following -e is the binary eccentricity; and the quantity following -m is the mass ratio. For fully detailed information, please see our paper, "A Machine Learns to Predict the Stability of Circumbinary Planets", out now on MNRAS and [arXiv](https://arxiv.org/abs/1801.03955).


# Tutorial (or: how to write our paper from scratch)

Please note some nonsensical comments or naming conventions may have survived in my hurriedly cleaned-up code. Please also note this code was written in Python 2 and before I knew of the existence of PEP-8. 

## Generating training data
With observational data from only a handful of circumbinary planets, we elected to generate the training data using the numerical integrator REBOUND. We begin by dropping the µ dimension, holding mass ratio constant at 0.1 while still varying semi-major axis, eccentricity, and initial phase. We build the training set with the following script in the /mu_10/ directory.  

```
python sim_10.py
```

In this script, we first set the hyperparameters of our simulation: here we use 10 phases per draw from [e, a] space and 1000 draws, where e is drawn (naively) uniformly from [0, 0.99] and a is drawn uniformly from a +/- 33% envelope surrounding the output of the function a<sub>crit</sub>(µ,e), described in Holman & Weigert (1999) as a sum of terms of products of µ and e up to the second power. Of course, for now µ is 0.1. For each of these 100,000 initial seedings, we use REBOUND's IAS15 integrator to simulate 100,000 binary periods. At each timestep during the simulation, REBOUND provides x, y, z, v<sub>x</sub>, v<sub>y</sub>, and v<sub>z</sub>. 

A seeding is labeled unstable if at any timestep v surpasses the escape velocity, or if the sampled initial semi-major axis is less than apoapsis, indicating the possibility of a crossed orbit. The latter case is checked before the costly simulation. If even one of the ten simulations per draw turn out unstable, we call the whole seeding unstable. 

I had output the labeled training data of ten thousand points (1 for stable, 0 for unstable) to .out files in batch jobs run on an HPC, but for this size of data you can certainly output to .txt or .csv files. The rest of this tutorial will assume the latter case and the code has been updated to reflect this.

You can replicate this exercise for different slices in mass ratio space to see how the islands of instability change.


## Building and training µ-constant DNN
We then build and train a neural network on the data with the following script. 

```
python make_model_10.py
```

In this script, we read in the seeding information (e and a expressed as a ratio of planet semi-major axis and binary semi-major axis, which, in another assumption within our simplified model, is 1.0) and fraction of samples per seeding found to be stable. We take the floor of this value to be our stability label. So far, all we've done was build towards a fancier model of what Holman and Wiegert did almost twenty years ago. 

Now we introduce another feature: how far away a planet is from the next lowest mean motion resonance. To do this we normalize the semi-major axis ratio against the semi-major axis predicted by the Holman-Wiegert formula given (µ=0.1, e), convert this re-parameterized semi-major axis ratio to a period ratio (ζ) using the Newtonian version of Kepler's third law, then use it to inform what we call the "resonance proxy", ε.

There is more than one way to do this, but here we simply take ε = 0.5 * (ζ - ⌊ζ⌋), where ⌊ζ⌋ is the floor of ζ, representing how far a planet is from the next largest integer. To see how far away a planet is from the next closest integer period ratio, one could also take ε = 0.5 - |ζ - ⌊ζ⌋ - 0.5|. 

We use sklearn to split the training data 0.75/0.25 between training/validation. We use the Keras library to build our deep neural network. Okay, so this is probably why you're here.

[Keras](https://keras.io/) helps you abstract away the code and think of DNNs as building blocks of customizable layers. We ended up choosing 6 layers each with 24 neurons and 20% dropout. The intermediary hidden layers use the rectified linear unit (ReLU) activation function, while the final output layer uses the sigmoid activation function. Our optimizer was rmsprop and our loss function was binary crossentropy. Our batch size was 120 and we trained the DNN for 100 epochs (one epoch is a back-and-forth pass through the network). 

We experimented with lots of different setups, and since we settled for the first configuration that outperformed Holman-Wiegert for precision, recall, and accuracy, you could probably beat us! Definitely check out the keras documentation for more ways you can design your neural network. There are certainly tons of hyperparameters to play with.


## Predicting with our trained DNN
Now we can make predictions and plot our results. But in order to do so we first need some test data. So we change the output file name to something like test_mu_10.txt and re-run sim_10.py. You can change the number of draws for a bigger or smaller test set.

For our paper we used LaTeX fonts, but we're not here to write a paper, so most of the formatting has been removed or commented out. The left panel of Figure 3 in our paper simply shows the unparameterized simulated data from REBOUND.

```
python unparam_simulated.py
```

Now let's actually make our predictions using our trained DNN and plot the results in reparameterized space. 

```
python use_model_10.py
```

All we do here is take test_mu_10.txt as input and, to save resources, run it through some checks rather than run it all through the trained DNN (but feel free to do the latter). Only if the sample is seeded within +/- 20% of the Holman-Wiegert critical threshold do we feed it into the model. Note the flags differentiating model- and heuristic-predicted outputs; we eventually elected not to use them but these can be useful when plotting results or troubleshooting. 

```
python reparam_preds.py
```

Here we plot the right panel of Figure 4, visualizing the islands of instability, false positives, and false negatives. We can get a better read on our model's performaance by comparing its accuracy, precision, and recall with that of the Holman-Wiegert model for an increasingly narrower band around the critical threshold. Far from this boundary, both models work reasonably well, but as we get towards the resonances and islands, we can see that the DNN begins to do much better. We create the accuracy, precision, and recall plots of Figure 5 with the following code. 

```
python figure5.py
```

## Graduating from slices
Great! Now that we've gone through the whole process for a slice of the data, let's step back outside the mu_10 folder and look at the wider breadth of mass ratios, (0, 0.5]. Some key differences this time: we train our DNN on a million data points (as we mentioned above, not really, but we'll get to that shortly); also, since we're adding another dimension, we can't plot in parameter space like we did before. In the repo are the original training set (big_job_narrow.txt) and test set (more_samples.txt), but the code has you making a million samples, if you're into cooking from scratch and setting your laptop on fire. 

In their basic configuration, Holman and Wiegert already figured out how to make good predictions for the stability of most circumbinary planets. The significant gains in our work come from our ability to generate better predictions about the edge cases - regions very close to Holman and Wiegert's analytic stability threshold, where mean motion resonances cause planets predicted by the analytic threshold as stable to be actually unstable, and vice versa. While in the mu_10 case we trained and tested on a +/- 33% envelope, in this more general case we train on a tighter envelope, +/- 20%, which was shown to encompass every Holman-Wiegert-predicted error given an independently generated 1e5-point test set. This way we could best exploit the DNN by training it only on what it needed to see. 

```
python sim.py
```

As you can see, the design and training of the DNN didn't change much, besides adding a µ column and doubling the number of neurons in each layer to 48. We toyed with many different architectures but decided to deviate from the mu_10 case as little as possible. You could (and we did) spend weeks testing countless variations of layer count, neuron count, dropout percentage, optimizer used, activation function. You could also realize knobs are made not to be fiddled with but to be ignored, and live a much happier life.

```
python make_model.py
```

Similarly for generating predictions on the test set, there isn't much difference outside the additional varying dimension. 

```
python use_model.py
```

Finally, we plot precision, recall, and accuracy against varying envelope sizes.

```
python figure5.py
```

Hopefully you'll see something similar to Figure 5 from our paper, or even better!


## Wishlist
- doing this all in a Jupyter notebook you can more easily follow
- doing this less verbosely
- introducing z coordinate (fifth and sixth degrees of freedom)
- abin != 1.0
- non-circular initial orbits
- more realistic sampling of eccentricities and mass ratios
- grad school

