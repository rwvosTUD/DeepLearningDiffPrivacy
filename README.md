


<h2>1. Introduction:</h2>
Recent explosive progress of the machine learning and deep learning industry raises questions about the privacy guarantess of this field. Due to its nature, training neural networks requires large and extensive datasets, which are mostly crowdsourced. They can often contain sensitive information which is why this topic is important . This blogpost is about reproducing the findings of the paper "Deep learning with Diferential privacy" by M. Abadi, A. Chu et al. Specificaly the diferential privacy mechanisms are combined with machine learning methods to train neural networks within single digit privacy budget. The next section gives insight into Differential privacy, which is followed by implementation of DP mechanisms. Section 4 presents the experimental setup of the network. Results and findings are presented in section 5.
<h2>2. Differential privacy</h2>
Differential privacy can be seen as a standard for privacy guarantee for algorithms that rely on data-bases. An algorithm is called differentialy private if by looking at the output, one cannot tell if certain individual data was included or not. This means that the behavior of the algorithm doesn't change (significantly) whether the inputs from one individual are or are not in the dataset. Consider two adjacent data bases, which means that they differ with at least a single entry. The output of the algorithm based on either of the databases shouldn't differ more than epsilon, to satisfy the delta,epsilon differential privacy.
<p><img src="https://miro.medium.com/max/700/1*IKhOQqMSkinUSGpD16XYpw.png" alt="Differential privacy" width="700" height="379" /></p>
<p style="text-align: center;"><em>Source: <a href="https://privacytools.seas.harvard.edu/files/privacytools/files/pedagogical-document-dp_new.pdf">Kobbi Nissim</a></em></p>


Mathematical definition of differential privacy is as follows: An algorithm K gives ε-differential privacy if for  data sets D and D′, differing by at most one row, and any S (input) ⊆ Range(K) 
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/s5Sz7xH/diffpr.png" alt="epsilon, delta differential privacy" width="439" height="60" /></p>

Epsilon is a metric of privacy loss, the lower the more privacy is guaranteed. Delta is an additional metric of probability of breaking the plain epsilon differential privacy.

Differential privacy deals with this problem by adding "noise"or randomness to the data, which prevents identifying any individual data points. Instead of returning the raw data, the algorithm return an approximation of the data. Intuitively, the noise level is related to the accuracy of the algorithm, the more noise we introduce, the higher the privacy, but the accuracy will be lower. Furthermore the gradient of the optimizer is clipped to prevent decrease output sensitivity to single input.
<h2>3. Differential privacy mechanisms:</h2>
<h3><span style="color: #99ccff; background-color: #ffffff;"><strong>2.1 DP-SGD</strong></span></h3>
The optimizer plays a vital role in the differential privacy framework. Through a series of steps it attempts to mask the incoming gradients in order to prevent them from becoming too informative with regard to respective samples. Furthermore, following a different interpretation this operation can also potentially fight overfitting to some degree. \\
The optimizing scheme works through four main steps. To start, the gradient based on the loss of every sample is obtained for every layer in the network using  the jacobian on a batch of samples. Once this is obtained this gradient is clipped, which is applied for each layer separately. This means that the gradients are normalized by the l2-norm scaled by a clipping constant C. Notice that clipping only occurs whenever l2-norm > C, otherwise the gradients pass through unmodified. After this all gradients per sample are combined into a single set to which noise is added. This noisy gradient is then used to update the parameters. These steps are included in their mathematical notation:  
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://iili.io/MwyHxt.jpg" alt="Pseudo code" width="491" height="254" /></p>

<h3><span style="color: #99ccff; background-color: #ffffff;"><strong>2.2 Moments accountant</strong></span></h3>
<p>The moment accountant keeps track of the pivacy loss encountered. Every time the optimization step is applied a small loss privacy in privacy is experienced. This loss can be computed and represented through either &epsilon or &delta. The main factor governing the magnitude of this loss is the noise level applied in the optimizer (&sigma)<p>. If this is known and epsilon is kept fixed (as in our experiments) the corresponding delta can be computed. The moments for the loss variable are obtained by an operation between the probability density functions of a Gaussian and a Gaussian mixture distribution. In practice, the values for E1 and E2 are approximated over a large sample space of 10k over the 2 distributions.

<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://iili.io/MwpybI.jpg" alt="Moment1" width="418" height="258" /></p>

This moment is computed for every possible lambda under consideration. For every iteration the respective moments are continuously summed using the composability theorem:

<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://iili.io/MwppON.jpg" alt="Moment2" width="216" height="85" /></p>

After this has been done the most limiting moment can be identified. Note that this might change between iterations, experical evidence shows a tendency to select higher moments first, yet this selection scales down to lower moments as the iterations continue. Using this moment the delta can be computed and checked against a threshold. As long as this threshold is not exceeded training continues. The corresponding delta can be obtained according to: 
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://iili.io/MwpmRp.jpg" alt="Moment3" width="282" height="84" /></p>

<h3><span style="color: #99ccff; background-color: #ffffff;"><strong>2.3 DP-PCA</strong></span></h3>
Principal component analysis is a strong tool used to approximate high dimensional data in fewer dimensions. Low dimensional approximation still preserves most of the information, at a smaller variable amount. This is beneficial for the duration of the training. 

The process of applying PCA to any data set start with standardazing the data. Since some inputs can be in range 0-1, others could be in range 0-100, which would result in bias towards the larger ranges. This is because PCA is very sensitive to variance in the variables. By standarizing the data all inputs will contribute equally to the result. This is the mathematical formulation :
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://builtin.com/sites/www.builtin.com/files/styles/ckeditor_optimize/public/inline-images/national/Principal%2520Component%2520Analysis%2520Standardization.png" alt="eq" width="263" height="54" /></p>

Next step is to calculate the covariance matrix of the data. Covariance matrix is a symmetrical matrix of size  equal to the input size. The entries are the covariances associated with the pairs of variables. The diagonal is composed of variances, as covariance of variable with itself is a variance. Covariance matrix can be obtained by multiplying the input data by the transpose of the data. 

To obtain a Differentially private version of the pca, one needs to add noise to the variance matrix. To preserve the properties of the covariance matrix, the added noise should be symmetrical aswell. For this reproduction we are using Gaussian noise as the authors of the paper suggested. Noise level can be varied by adjusting the standard deviation of the distribution. Following code snippet presents how to obtain private covariance matrix:
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/N2rNr18/noise.png" alt="noise" width="1064" height="171" /></p>

By calculating the eigenvalues and eigenvectors of the noisy covariance matrix, one can identify the principal components of the data set. Ordering the eigenvectors in descending order corresponds to ordering the components in order of significance. To obtain n-dimensional projection a feature vectors is constructed, containing n first sorted eigenvectors. 

The final projection is obtained by reorienting the input data along the calculated principal axis. This can be done by multiplying the transposes of the feature vector and the input data.

<h2>4. Reproduced setup:</h2>
<h3><span style="color: #99ccff; background-color: #ffffff;"><strong>4.1 MNIST</strong></span></h3>
The data set used to reproduce the findings of the paper was MNIST dataset which is a publicly sourced dataset containing 70 000 of 28 by 28 pixel images of hand written digits along with the corresponding labels from 0 to 9. The data set is loaded from the Tensorflow/Keras library as indicated in the paper. This data set is widely available which is an advantage for the reproducers. The data set is split into a training and test set in ratio 6:1. 
<h3><span style="color: #99ccff; background-color: #ffffff;"><strong>4.2 Preprocessing</strong></span></h3>
The data set comes in format of 3 dimensional array [n_samples,x_pixels,y_pixels], but for the purpose of training it has to be reshaped to 2 dimensional array [n_samples,x*y pixels]. After that the inputs are standardized as a part of preparation for applying the DP-PCA. The standardized data is divided into batches with the specified lot size of 600. 
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/rxjbH4G/batching.png" alt="batching" width="996" height="246" /></p>

<h3><span style="color: #99ccff; background-color: #ffffff;"><strong>4.3 Architecture</strong></span></h3>
The architecture of the network mentioned in the paper is simple, because it only consists of a input layer, single hidden layer with 1000 neurons and Relu activation followed by an output layers of size 10 with softmax activation.
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/QC27qZp/model.png" alt="model" width="859" height="248" /></p>

Unfortunately, the standard sequence model framework from Tensorflow cannot be used as applying the differential privacy mechanism requires inspecting and clipping the gradients, which breakes the parallel nature of the Tensorflow module. Thats why the training must be performed by "hand"by means of for loops. Following code snippet represents training of the neural network for one epoch, using the DP_SGD optimizer and moments accountant:
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/JHy6pLR/trainepoch.png" alt="trainepoch" width="1049" height="580" /></p>


<h2>5. Results:</h2>
The paper contained most of the information required to reproduce the results, although the source code is not directly available. From our perspective the hardest part to reproduce was the moment accountant. There seems to be a mistake in the moment accountant vs composition theorem privacy budget estimation. The figure 2 of the paper clerly indicates epsilon of 4.5 for 400 Epochs, while the paper reports 2.55 as the privacy loss estimation. This could be an editing error. 

The paper also indicates that the noisy covariance matrix that is used to find principal components of the batch input, is based on a random sample of data. For our reproduction, basing the covariance matrix on only a small sample of data resulted in very poor final accuracy. Only when the noisy covariance matrix was based on full training data set, which is 60 000 samples, we could obtain satisfying results.Furthermore, the authors mention taking the privacy cost of the Principal component analysis into account, but the method or quantitive formulation is not included. 

The goal was to reproduce the Figure 3 from the paper, which visualises the accuracy and privacy loss versus epochs for different noise levels. Below our results are presented. The test set accuracy is 89%, 95% and 96%  for (0.5, 10−5), (2, 10−5),and (8, 10−5)-differential privacy respectively.


<p><img style="float: left;" src="https://i.ibb.co/41PtsLL/highnoise.png" alt="highnoise" width="394" height="295" /><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/kVZLhhZ/mediumnoise.png" width="394" height="295" /></p>
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/gM6NfRX/lownoise.png" alt="Low noise" width="394" height="295" /></p>




Furthermor variation of number of hidden layers is conducted and similar results are obtained.
<p><img style="display: block; margin-left: auto; margin-right: auto;" src="https://i.ibb.co/xqC2xMn/hiddenlayers-varying.jpg" alt="" /></p>


<p>&nbsp;</p>
<h2>6. Conclusion:</h2>
