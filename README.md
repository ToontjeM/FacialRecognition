# FacialRecognition
Facial Recognition Using Postgresql &amp; PGvector

The objective of this experiment is to leverage the CLIP model in conjunction with PostgreSQL, employing the pgvector extension and PL/Python to execute transformation functions directly within the database for efficient searching. This setup involves a dataset of 202k low-resolution images. 

Instead of storing the images directly in the database, we store only their full file paths. The actual data stored in the database consists of image embeddings, which are generated by the CLIP model and encapsulated in 512-dimensional vectors as required by the model. This approach enables rapid search capabilities on a standard laptop.

We are showing also text to image search, searching on images content passing text as input. 

## Sample Dataset
I used the following
https://drive.google.com/drive/folders/0B7EVK8r0v71pTUZsaXdaSnZBZzg?resourcekey=0-rJlzl934LzC-Xp28GeIBzQ. The dataset contains more than 202k Images. Size of each image is < 20k.
Download the dataset and then unzip and tar xvf the file. It will create a directory with 202k images in one single directory. 
File : img_align_celeba.zip, 1.44GB

##CAUTIOUS: Please validate that there is no virus on the file. I did not check also all images in term of content. 

Use the script distribute_images.sh to create a directory with up to 250 images. It should take few minutes and  create 811 directories. 
This file is located inside Dataset folder

## Install all dependencies for transformers and validate that CLIP model is available

Postgresql 16 installed.

EDB Language pack installed.

Run pip install from EDB Python directory as: /Library/edb/languagepack/v4/Python-3.11/bin/pip install -r requirements.txt

Install pgvector 0.6 extension from https://github.com/pgvector/pgvector

Validate that pl-python3u is working well 

run select public.test_plpython() inside the database;

postgres=# select public.test_plpython();
     test_plpython     
 PL/Python is working!
(1 row)

Python Environment: The Python environment accessible to PostgreSQL should have the necessary libraries installed: 
After test that this program is working: 

%python test_clip.py

## Create Table, install Pl_python3u functions

Open psql and create the table 

drop table if exists public.pictures;

CREATE TABLE IF NOT EXISTS public.pictures ( id serial, imagepath text, tag text, embeddings vector(512) ) TABLESPACE pg_default;


Install the 3 functions inside DDL folder:

1 - generate_embeddings_clip_bytea -- Generate embedding from Bytea (Used by the streamlit application to get the input image and return an embedding to search inside the database)

2 - scan_specific_path_and_load -- Main function to generate embedding in batch and load data inside pictures table. 

3 - process_images_and_store_embeddings_batch -- Function to load folder of images, take an input pattern and call scan_specific_path_and_load for each folder in which we have images




## Load Data

You can load images from a directory directly calling the pl-python function: process_images_and_store_embeddings_batch(source_dir text,tag text,batch integer)

if you have images for instance on: /Users/francksidi/Downloads/celebrity/image_group_811

postgres#= select process_images_and_store_embeddings_batch(' /Users/francksidi/Downloads/celebrity/image_group_811', 'person', 32);

NOTICE:  Total processing time: 4.72 seconds
NOTICE:  Total image processing time: 2.40 seconds
NOTICE:  Total database insertion time: 0.03 seconds
NOTICE:  Total number of images processed: 99
process_images_and_store_embeddings_batch 
99
(1 row)


## Load All Data

You can load all data using the pl-python function from psql. we load in multiple phases as we did not create autonomous transactions.

postgres#= SET work_mem = '512MB';

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_1*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_2*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_3*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_4*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_5*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_6*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_7*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_8*','person',32);

select public.scan_specific_path_and_load('/Users/francksidi/Downloads/celebrity/image_group_9*','person',32);

## Similarity Search using sample python program

Change the connection info inside search.py

% python search.py '/users/francksidi/downloads/profile.jpg'

Fetching vector took 3.3727 seconds.

Querying similar images took 0.8264 seconds.

ID: 394872, ImagePath: /Users/francksidi/Downloads/celebrity/image_group_494/123370.jpg, Similarity: 0.38990872249157593

ID: 440568, ImagePath: /Users/francksidi/Downloads/celebrity/image_group_64/015999.jpg, Similarity: 0.3897554537178016

ID: 435829, ImagePath: /Users/francksidi/Downloads/celebrity/image_group_589/147039.jpg, Similarity: 0.35878975781406686

ID: 396142, ImagePath: /Users/francksidi/Downloads/celebrity/image_group_405/101064.jpg, Similarity: 0.35226602687072894

ID: 412995, ImagePath: /Users/francksidi/Downloads/celebrity/image_group_538/134483.jpg, Similarity: 0.34490723691677183


Now Create index on the table to speed up the search: 

The search is now moving from 0.8 sec to 0.08 sec. 


## Similarity Search using Streamlit application enabling WebCAM

Change the connection info inside streamlit_face_reco.py
Run from the command line. Copy the logo.png image in the directory in which the python program is running.

%streamlit run streamlit_face_reco.py

## Similarity Search using Streamlit application enabling WebCAM and Text Search on Images. 

For instance look for images from a text description: old man, old lady, blond lady ,...

%streamlit run streamlit_face_reco_text.py


## Lesson Learned 

### Model Pre-Loading

To optimize performance, it's crucial to cache the model on the SD card, preventing the need to reload it with each execution—a process that typically takes between 2 to 5 seconds. Ideally, the model should load once at the beginning of a session. An enhancement would be to preload the model for all sessions right at the server startup, ensuring immediate availability and further efficiency gains.

### Define the model and processor outside the loop to avoid reloading them for each image
	model_name = "openai/clip-vit-base-patch32"
	if 'model' not in SD:
		SD['model'] =  CLIPModel.from_pretrained(model_name)
		SD['processor'] = CLIPProcessor.from_pretrained(model_name)
		plpy.notice("Model & Processor Loaded")
	else:
		plpy.notice("Model & Processor Reused")
	model = SD['model']
	processor = SD['processor']
	
### Batching Images Processing

Using a batch approach for images processing allowed me to triple the throughput moving from 12.5 Images/sec to 40 images per sec using a batch of 24. 

### Transactions in Python Program 

From now, we are using 1 single transaction to load the data and process embeddings. We should use autonomous transactions to split with multiple transactions and call an autonomous transaction for each call to: public.process_images_and_store_embeddings_batch. 

### Similarity Search

It’s important not to pass any function to a query to process the embedding as the function will be called for each row of the table. I limit below just for 10 rows. We can see that we are calling the function for each row. 

Don’t do something like this: 

postgres=# explain analyze SELECT id, imagepath, 1 - (embeddings <-> public.generate_embeddings_clip('/users/francksidi/downloads/profile.jpg', 'person')) as similarity

FROM (select * from pictures limit 10) a 

ORDER BY similarity DESC Limit 3 ;

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

NOTICE:  Model & Processor Reused

### Efficiency of Index

Adding HSNW index improved the search query by 10X as we moved from 0.8sec to 0.08 sec. 

                                                                     
