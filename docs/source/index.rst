.. doct documentation master file, created by


DeepPurpose documentation!
================================
Welcome! This is the documentation for DeepPurpose, a PyTorch-based deep learning library for Drug Target Interaction.
The Github repository is located `here <https://github.com/kexinhuang12345/DeepPurpose>`_.


1 How to Start
--------------



1.1 Download
^^^^^^^^^^^^

.. code-block:: bash

   $ git clone git@github.com:kexinhuang12345/DeepPurpose.git
   $ ###  Download code repository 
   $
   $
   $ cd DeepPurpose
   $ ### Change directory to DeepPurpose 


1.2 Installation
^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   $ conda env create -f env.yml  
   $ ## Build virtual environment with all packages installed using conda
   $ 
   $ conda activate DeepPurpose
   $ ##  Activate conda environment
   $
   $
   $ conda deactivate ### exit




2 Run
--------------------------






3 Documentation
--------------------------



3.1 Encoder Models
^^^^^^^^^^^^^^^^^^^^



**environment** 
.. code-block:: python

	import torch
	from torch.autograd import Variable
	import torch.nn.functional as F
	from torch.utils import data
	from torch.utils.data import SequentialSampler
	from torch import nn 

	from tqdm import tqdm
	import matplotlib.pyplot as plt
	import numpy as np
	import pandas as pd
	from time import time
	from sklearn.metrics import mean_squared_error, roc_auc_score, average_precision_score, f1_score
	from lifelines.utils import concordance_index
	from scipy.stats import pearsonr
	import pickle 
	import copy
	from prettytable import PrettyTable
	import scikitplot as skplt



.. code-block:: python
	
	DeepPurpose.models.transformer(nn.Sequential)

`Transformer <https://www.analyticsvidhya.com/blog/2019/06/understanding-transformers-nlp-state-of-the-art-models/>`_ can be used to encode both drug and protein. 


.. code-block:: python

	def __init__(self, encoding, **config):
		super(transformer, self).__init__()
		if encoding == 'drug':
			self.emb = Embeddings(config['input_dim_drug'], config['transformer_emb_size_drug'], 50, config['transformer_dropout_rate'])
			self.encoder = Encoder_MultipleLayers(config['transformer_n_layer_drug'], 
														config['transformer_emb_size_drug'], 
														config['transformer_intermediate_size_drug'], 
														config['transformer_num_attention_heads_drug'],
														config['transformer_attention_probs_dropout'],
														config['transformer_hidden_dropout_rate'])
		elif encoding == 'protein':
			self.emb = Embeddings(config['input_dim_protein'], config['transformer_emb_size_target'], 545, config['transformer_dropout_rate'])
			self.encoder = Encoder_MultipleLayers(config['transformer_n_layer_target'], 
														config['transformer_emb_size_target'], 
														config['transformer_intermediate_size_target'], 
														config['transformer_num_attention_heads_target'],
														config['transformer_attention_probs_dropout'],
														config['transformer_hidden_dropout_rate'])


* **encoding** (string, "drug" or "protein") - specify input type, "drug" or "protein". 

* **config** (kwargs, keyword arguments) - specify the parameter of transformer. 



.. code-block:: python

	def forward(self, v):
		e = v[0].long().to(device)
		e_mask = v[1].long().to(device)
		ex_e_mask = e_mask.unsqueeze(1).unsqueeze(2)
		ex_e_mask = (1.0 - ex_e_mask) * -10000.0

		emb = self.emb(e)
		encoded_layers = self.encoder(emb.float(), ex_e_mask.float())
		return encoded_layers[:,0]

* **v** (torch.Tensor) - input feature of transformer. 









~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~









.. code-block:: python

	class DeepPurpose.models.CNN(nn.Sequential)


`CNN (Convolutional Neural Network) <https://en.wikipedia.org/wiki/Convolutional_neural_network>`_ can be used to encode drug. 


.. code-block:: python

	def __init__(self, encoding, **config):
		super(CNN, self).__init__()
		if encoding == 'drug':
			in_ch = [63] + config['cnn_drug_filters']
			kernels = config['cnn_drug_kernels']
			layer_size = len(config['cnn_drug_filters'])
			self.conv = nn.ModuleList([nn.Conv1d(in_channels = in_ch[i], 
										out_channels = in_ch[i+1], 
										kernel_size = kernels[i]) for i in range(layer_size)])
			self.conv = self.conv.double()
			n_size_d = self._get_conv_output((63, 100))
			#n_size_d = 1000
			self.fc1 = nn.Linear(n_size_d, config['hidden_dim_drug'])

		if encoding == 'protein':
			in_ch = [26] + config['cnn_target_filters']
			kernels = config['cnn_target_kernels']
			layer_size = len(config['cnn_target_filters'])
			self.conv = nn.ModuleList([nn.Conv1d(in_channels = in_ch[i], 
												out_channels = in_ch[i+1], 
												kernel_size = kernels[i]) for i in range(layer_size)])
			self.conv = self.conv.double()
			n_size_p = self._get_conv_output((26, 1000))

			self.fc1 = nn.Linear(n_size_p, config['hidden_dim_protein'])

* **encoding** (string, "drug" or "protein") - specify input type, "drug" or "protein". 

* **config** (kwargs, keyword arguments) - specify the parameter of transformer. 



.. code-block:: python


	def _get_conv_output(self, shape):
		bs = 1
		input = Variable(torch.rand(bs, *shape))
		output_feat = self._forward_features(input.double())
		n_size = output_feat.data.view(bs, -1).size(1)
		return n_size

	def _forward_features(self, x):
		for l in self.conv:
			x = F.relu(l(x))
		x = F.adaptive_max_pool1d(x, output_size=1)
		return x

	def forward(self, v):
		v = self._forward_features(v.double())
		v = v.view(v.size(0), -1)
		v = self.fc1(v.float())
		return v

* **v** (torch.Tensor) - input feature of CNN. 


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


.. code-block:: python

	class DeepPurpose.models.CNN_RNN(nn.Sequential)

CNN+RNN is 

.. code-block:: python

	def __init__(self, encoding, **config):
		super(CNN_RNN, self).__init__()
		if encoding == 'drug':
			in_ch = [63] + config['cnn_drug_filters']
			self.in_ch = in_ch[-1]
			kernels = config['cnn_drug_kernels']
			layer_size = len(config['cnn_drug_filters'])
			self.conv = nn.ModuleList([nn.Conv1d(in_channels = in_ch[i], 
													out_channels = in_ch[i+1], 
													kernel_size = kernels[i]) for i in range(layer_size)])
			self.conv = self.conv.double()
			n_size_d = self._get_conv_output((63, 100)) # auto get the seq_len of CNN output

			if config['rnn_Use_GRU_LSTM_drug'] == 'LSTM':
				self.rnn = nn.LSTM(input_size = in_ch[-1], 
								hidden_size = config['rnn_drug_hid_dim'],
								num_layers = config['rnn_drug_n_layers'],
								batch_first = True,
								bidirectional = config['rnn_drug_bidirectional'])
			
			elif config['rnn_Use_GRU_LSTM_drug'] == 'GRU':
				self.rnn = nn.GRU(input_size = in_ch[-1], 
								hidden_size = config['rnn_drug_hid_dim'],
								num_layers = config['rnn_drug_n_layers'],
								batch_first = True,
								bidirectional = config['rnn_drug_bidirectional'])
			else:
				raise AttributeError('Please use LSTM or GRU.')
			self.rnn = self.rnn.double()
			self.fc1 = nn.Linear(config['rnn_drug_hid_dim'] * config['rnn_drug_n_layers'] * n_size_d, config['hidden_dim_drug'])

		if encoding == 'protein':
			in_ch = [26] + config['cnn_target_filters']
			self.in_ch = in_ch[-1]
			kernels = config['cnn_target_kernels']
			layer_size = len(config['cnn_target_filters'])
			self.conv = nn.ModuleList([nn.Conv1d(in_channels = in_ch[i], 
													out_channels = in_ch[i+1], 
													kernel_size = kernels[i]) for i in range(layer_size)])
			self.conv = self.conv.double()
			n_size_p = self._get_conv_output((26, 1000))

			if config['rnn_Use_GRU_LSTM_target'] == 'LSTM':
				self.rnn = nn.LSTM(input_size = in_ch[-1], 
								hidden_size = config['rnn_target_hid_dim'],
								num_layers = config['rnn_target_n_layers'],
								batch_first = True,
								bidirectional = config['rnn_target_bidirectional'])

			elif config['rnn_Use_GRU_LSTM_target'] == 'GRU':
				self.rnn = nn.GRU(input_size = in_ch[-1], 
								hidden_size = config['rnn_target_hid_dim'],
								num_layers = config['rnn_target_n_layers'],
								batch_first = True,
								bidirectional = config['rnn_target_bidirectional'])
			else:
				raise AttributeError('Please use LSTM or GRU.')

			self.rnn = self.rnn.double()
			self.fc1 = nn.Linear(config['rnn_target_hid_dim'] * config['rnn_target_n_layers'] * n_size_p, config['hidden_dim_protein'])
		self.encoding = encoding
		self.config = config

* **encoding** (string, "drug" or "protein") - specify input type, "drug" or "protein". 
* **config** (kwargs, keyword arguments) - specify the parameter of transformer. 

.. code-block:: python
	forward(self, v)

* **v**


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

	class DeepPurpose.models.MLP(nn.Sequential)


Multi-Layer Perceptron

.. code-block:: python

	__init__(self, input_dim, hidden_dim, hidden_dims):

* **input_dim** (int) - dimension of input feature. 
* **hidden_dim** (int) - dimension of hidden layer. 


.. code-block:: python

	forward(self, v)

* **v**

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



.. code-block:: python

	class DeepPurpose.models.MPNN(nn.Sequential)

Message Passing Neural Network 


.. code-block:: python

	__init__(self, mpnn_hidden_size, mpnn_depth) 



* **mpnn_hidden_size** (int) - dimension of hidden layer in MPNN. 
* **mpnn_depth** (int) - depth of MPNN. 


.. code-block:: python

	forward(self, feature)

* **feature** (tuple of length 5)


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~



.. code-block:: python

	class DeepPurpose.models.Classifier(nn.Sequential)

Classifier 

.. code-block:: python

	__init__(self, model_drug, model_protein, **config) 


* **model_drug** (DeepPurpose.models.XX) - Encoder model for drug. XX can be "transformer", "MPNN", "CNN", "CNN_RNN" ..., 
* **model_protein** (DeepPurpose.models.XX) - Encoder model for protein. XX can be "transformer", "CNN", "CNN_RNN" ..., 
* **config** (kwargs, keyword arguments) - specify the parameter of classifier.  




.. code-block:: python

	forward(self, v_D, v_P)


* **v_D** 
* **v_P** 



~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: python

	class DeepPurpose.models.DBTA

DBTA 

.. code-block:: python

	__init__(self, **config)

* **config** (kwargs, keyword arguments) - specify the parameter of classifier.  
















