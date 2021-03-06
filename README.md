# Clockwork RNNs 

Clockwork RNNs are a set of simple RNNs that run at different clock speeds but share their hidden states and produce a common output. 

Suppose there are `n` RNNs or modules `{RNN1, . . ., RNNn}` and each RNN is assigned a clock period `T ∈ { T1 , . . . , Tn }`. `RNNi` is active only if the current time step `t % Ti == 0` Each RNN is internally fully interconnected, but the recurrent connections from `RNNj` to `RNNi` exists only if the period `Ti` is smaller than period `Tj`. 

## Observations and Modifications
Intuitively, the slower RNNs (the ones with larger periods) act as shortcut pathways for information and gradients between large number of timesteps. The faster RNNs (the ones with shorter periods) handle short term, local operations. 

Taking the above logic further, we can imagine that, for a given task, the slower RNN comes in every once in a while, puts in the instructions and then hibernates till it is again active. The slower modules take these instructions and operate on them at every timestep to get the desired output. CWRNN works really well for this setting the of simple generation (which is the premise of the 1st experiment that the CWRNN was trained on in the original paper Section 4.1).

However, when looked from a discriminative perspective, the way connections are made make less sense. The faster modules are seeing the inputs and are making short term decisions or keeping important information in their states. The slower modules that come in later should be able to see these states of faster modules to understand what has occured in the intervening timesteps when it was not active. In other words, we cannot expect a perticular change in input at a perticular timestep in which the slower module becomes active. The observations made by faster module should be summarized by the slower module and stored in the slower module's hidden state. The faster module, due to its shorter memory will forget this information eventually, but can access this since the slower modules have it in their hidden states.

This is much better understood in the example context of Language Modelling(LM). Suppose we have a character level LM, with CWRNN. The faster module, is seeing everything - opening or closing of parenthesis, punctuation, the gender of the subject, the tone of the sentence, the tense of the text / sentences. The slower modules should come in and should be able to retrive all this information, as that is only available to the fastest module. And then the slower module should store it in its hidden state in a some form. It can then make it available for the faster modules at every one of their active timesteps. Seeing one character for every 16 characters in a text sequence can tell nothing about the text.

## Modified CWRNN

CWRNN are similar to Hierarchical Subsampling Networks (HSNs) and Deep RNNs except that subsampling is done implicity. 

The Deep RNN from Graves 2013 - Generating Sequences with RNNs
![deeprnn](https://cloud.githubusercontent.com/assets/8753078/11612816/9bddf020-9c2e-11e5-850e-0b50381b6d5b.png)

HSNs are similar to Deep RNNs but RNNs at higher layers subsample the outputs of the layer below, before taking them as inputs. This is done with weighted scheme
![hsn](https://cloud.githubusercontent.com/assets/8753078/11612818/9c901016-9c2e-11e5-8b7f-8a668afd0cd7.png)

Suppose we have a CWRNN with 6 modules with periods `[1, 2, 4, 8, 16]`. The first module is fully interconnected with all other 5 modules and itself. The remaining modules are connected with themselves completely and every module before them, this gives roughly a combination of Hierarchical Subsampling Networks (HSNs) and Deep RNNs. Note that the connection scheme mentioned for the 4 modules is opposite of what is proposed in the paper. 

By the notations given in the paper, this would mean:
![depp](https://cloud.githubusercontent.com/assets/8753078/11612805/453c4b86-9c2e-11e5-9b0d-52dbab1c005a.png)

Keeping with the reasons provided in the previous section, we can remove the connections from input to all slower modules. Just providing the input to the first module is sufficient.

Further, if we have symmetric clock rates with the blocks being active in a perticular pattern, such as the one shown below, we could get networks that resemble the Seq2Seq Architectures of Sutskever et al. 
![seq2seq](https://cloud.githubusercontent.com/assets/8753078/11612806/46e59834-9c2e-11e5-8309-7a93aa72383c.png)

This means that CWRNNs, in a way subsume a wide range of Stacked and Multiscale RNN achitectures.

The modified recurrent weight matrix supports long term memory. Here are 2 images that capture the long term properties of original cwrnn vs the modified one:
Both plots are Gradient Norm vs Time Step: 

Original implementation gives:
![grads_old](https://cloud.githubusercontent.com/assets/8753078/11612856/4f2cb75a-9c30-11e5-8ca3-815b43b6698a.png)

Modified - HSN/Deep RNN like:
![grads_new](https://cloud.githubusercontent.com/assets/8753078/11612855/4e8db6d2-9c30-11e5-82ff-f9ff91520851.png)

#### Other Extensions:

* **Round Robin RNNs**: The fastest modules (usually ones with period 1) are seeing everything. Instead that  can be split up into more modules, such that modules take turns one after that other to be active. This should help with vanishing gradients. **RESULTS**: This is cheating in some sense - it helps in keeping the information, but it as good as using a single RNN and storing the hidden states explicitly at regular intervals.
* **Dynamic Forgetting during training**: Forget hidden state very frequently during inital stages of training, but extend the period of forgetting over time.
* **Various Clock Periods**: RESULTS: Not much difference. Exponential series of 2 is fine. 
* 
	* Symmetric: Has given best results so far, within 		the original CWRNN model. Maybe due to the reason 		stated above.
	* Fibonacci (Virahanka)
	* Different Exponential Series
	* Random
* **Dropout**: RESULTS: Helps, but slows down training a lot
* **Other Fancy Activations**
	
## Results

~~Initial Experiments have shown that:~~

Upon studying the gradient norm, it was stupid to start the main clock at 1, instead of 0. Thus all experiments from the past 12 days or more - according to the Github commit is useless.

**Current Best Result**: Single Layer Clockwork RNN with Symmetric Clock Periods, with about 1.3 million parameters, without dropout gave test CE loss of about **1.4**, which equals to about **2 BPC**. Current state of the art, which I belive is the Grid LSTM paper from Google DeepMind reported BPC of **1.47**.



