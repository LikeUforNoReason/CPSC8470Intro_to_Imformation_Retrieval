# ConversationQueryRewriter


## Dependencies

We require python >= 3.6, pip >= 20.1.1, cuda >= 9, pytorch, transformers 2.3.0, apex, scikit-learn, and a handful of other supporting libraries. 

In order to avoid that a python installation may not meet the requirements of each application, we create a virtual environment in which a specific python version is installed, as well as many other packages. To create a virtual environment, determine the directory where you want to place it, and run the venv module as a script in the directory path: 
```
python3 -m venv tutorial-env
```
After creating the virtual environment, we need to activate it. 
On Windows, run: 
```
tutorial-env\Scripts\activate.bat
```
On Unix or MacOS, run: 
```
source tutorial-env/bin/activate
```
Activating the virtual environment will change the prompt of the terminal you are using to display the virtual environment you are using, and modify the environment so that the python command will run the specific version of Python that has been installed.

To install dependencies use

```
pip install -r requirements.txt
git clone https://github.com/NVIDIA/apex
cd apex
pip install -v --no-cache-dir ./
```

The spaCy model for English is needed and can be fetched with:

```
python -m spacy download en_core_web_sm
```

The easiest way to run this code is to use:

```
export PYTHONPATH=${PYTHONPATH}:`pwd`
```

## Data

By default, we expect source and preprocessed data to be stored in "./data".

### TREC CAsT 2019 Data

TREC CAsT 2019 data can be obtained from [here](https://github.com/daltonj/treccastweb).

Use the following commands to download the TREC CAsT 2019 data to the `data` folder:

```
cd data
wget https://raw.githubusercontent.com/daltonj/treccastweb/master/2019/data/evaluation/evaluation_topics_v1.0.json https://raw.githubusercontent.com/daltonj/treccastweb/master/2019/data/evaluation/evaluation_topics_annotated_resolved_v1.0.tsv
```

We added some other behaviors in this system, such as providing clarifying questions, guiding the dialogue to a
certain extent, and so on. So we modified the downloaded TREC CAsT 2019 data set, and you can directly use our modified data located in `data`.

### TREC CAsT 2020 Data

TREC CAsT 2020 data is very different from TREC CAsT 2019 data. The TREC CAsT 2020 data set has new data format.  Topics will unfold through the interactions. And a system response will be provided with each previous turn. It can be obtained from [here](https://github.com/daltonj/treccastweb).

Use the following commands to download the TREC CAsT 2020 data to the `data` folder or you can directly use our downloaded data located in `data/TREC_CAsT_2020_Data`:

```
cd data
wget https://raw.githubusercontent.com/daltonj/treccastweb/master/2020/2020_manual_evaluation_topics_v1.0.json https://raw.githubusercontent.com/daltonj/treccastweb/master/2020/2020_automatic_evaluation_topics_v1.0.json
```

### MS MARCO Conversatioanl Search Corpus

MS MARCO Conversational Search corpus is used to genearte weak supervison data and it can be obtained from [here](https://github.com/microsoft/MSMARCO-Conversational-Search).

Use the following commands to get and unpack the Dev sessions:

```
mkdir data/ms_marco
cd data/ms_marco
wget https://msmarco.blob.core.windows.net/conversationalsearch/ann_session_dev.tar.gz
tar xvzf ann_session_dev.tar.gz
```

### Preprocess modified TREC CAsT 2019 Data

Run `cqr/preprocess.py` to convert and split folds for modified TREC CAsT 2019 data:

```
python cqr/preprocess.py
```

This will generate `eval_topics.jsonl` in `data` along with 5 folds `eval_topics.jsonl.x(x=0,1,2,3,4)` for cross-validation.

### Preprocess TREC CAsT 2020 Data

If you want to use TREC CAsT 2020 as your module tranning data set, you can run `cqr/modified_preprocess.py` to convert and split folds for TREC CAsT 2020 data:

```
python cqr/modified_preprocess.py
```

This will generate `eval_topics.jsonl` in `data` along with 5 folds `eval_topics.jsonl.x(x=0,1,2,3,4)` for cross-validation.

## Generate Weak Supervision Data

You can directly use our generated files located in `data/weak_supervision_data`.


## Train

Our models can be trained by:

```
python cqr/run_training.py --model_name_or_path <pretrained_model_path> --train_file <input_json_file> --output_dir <output_model_path>
```

### Cross-validation on TREC CAsT data set

For example:

```
nohup python -u cqr/run_training.py --output_dir=models/query-rewriter-cv-bs2-e4 --train_file data/eval_topics.jsonl --cross_validate --model_name_or_path=gpt2-medium --per_gpu_train_batch_size=2 --num_train_epochs=4 --save_steps=-1 &> run_train_query_rewriter_cv.log &
```

You would get 5 models (e.g. models/model-medium-cv-s2-e4-\<i\> where i = 0..4) using the default setting (NUM\_FOLD=5).

### Rule-based

For example:

```
nohup python -u cqr/run_training.py --output_dir=models/query-rewriter-rule-based-bs2-e1 --train_file data/weak_supervision_data/rule-based.jsonl --model_name_or_path=gpt2-medium --per_gpu_train_batch_size=2 --save_steps=-1 &> run_train_query_rewriter_rule_based.log &
```

### Self-learn

Recall that we have 5 sets of data generated by 5 different simplifiers trained with different training data. Query rewriting models could be trained each using data from one query simplifier. For example:

```
nohup python -u cqr/run_training.py --output_dir=models/query-rewriter-self-learn-bs2-e1-<i> --train_file data/weak_supervision_data/self-learn.jsonl.<i> --model_name_or_path=gpt2-medium --per_gpu_train_batch_size=2 --save_steps=-1 &> run_train_query_rewriter_self_learn_<i>.log &
```

where i = 0, 1, ..., 4.

### Rule-based + CV

Just change the parameter 'model_name_or_path' in the cross-validation example from a pretrained GPT-2 to the directory of the trained rule-based model. For example:

```
nohup python -u cqr/run_training.py --output_dir=models/query-rewriter-rule-based-bs2-e1-cv-e4 --train_file data/eval_topics.jsonl --cross_validate --model_name_or_path=models/query-rewriter-rule-based-bs2-e1 --per_gpu_train_batch_size=2 --save_steps=-1 &> run_train_query_rewriter_rule_based_plus_cv.log &
```

### Self-learn + CV

Don't forget to use '--init_from_multiple_models' in this setting to start from 5 models trained on 5 different sets of weak supervision data. For example:

```
nohup python -u cqr/run_training.py --output_dir=models/query-rewriter-self-learn-bs2-e1-cv-e4 --train_file data/eval_topics.jsonl --cross_validate --init_from_multiple_models --model_name_or_path=models/query-rewriter-self-learn-bs2-e1 --per_gpu_train_batch_size=2 --save_steps=-1 &> run_train_query_rewriter_self_learn_plus_cv.log &
```


## Inference

You can use the following command to do inference:

```
python cqr/run_prediction.py --model_path <model_path> --input_file <input_json_file> --output_file <output_json_file>
```

### Cross-validation

For example:

```
python cqr/run_prediction.py --model_path=models/query-rewriter-cv-bs2-e4 --cross_validate --input_file=data/eval_topics.jsonl --output_file=cv-predictions.jsonl
```

### Rule-based

For example:
```
python cqr/run_prediction.py --model_path=models/query-rewriter-rule-based-bs2-e1 --input_file=data/eval_topics.jsonl --output_file=rule-based-predictions.jsonl
```

### Self-learn

Recall that we have 5 models trained on 5 different sets of generated data. Thus we need the `--cross_validate` option to do the inference of their unseen parts:

```
python cqr/run_prediction.py --model_path=models/query-rewriter-self-learn-bs2-e1 --cross_validate --input_file=data/eval_topics.jsonl --output_file=model-based-predictions.jsonl
```

### Rule-based + CV

For example:

```
python cqr/run_prediction.py --model_path=models/query-rewriter-rule-based-bs2-e1-cv-e4 --cross_validate --input_file=data/eval_topics.jsonl --output_file=rule-based-plus-cv-predictions.jsonl
```

### Self-learn + CV

For example:
```
python cqr/run_prediction.py --model_path=models/query-rewriter-self-learn-bs2-e1-cv-e4 --cross_validate --input_file=data/eval_topics.jsonl --output_file=model-based-plus-cv-predictions.jsonl
```


