# Homecraft Retail lab script: build an e-commerce search bar with Elastic ESRE and Google's GenAI

This is the step-by-step guide for enablement hands-on labs, and refers to the code repo https://github.com/valerioarvizzigno/homecraft_vertex


## Configuration steps

1. Sign-up to a [free trial account](https://www.elastic.co/cloud/elasticsearch-service/signup) of Elastic (or alternatively subscribe via GCP MP)
2. Setup your Elastic cluster:
   - select a GCP region, the latest Elastic version, Autoscaling set to "None"
   - 1-zone 360GB Hot node
   - 1-zone 4GB Machine Learning node
   - 1-zone 4GB Kibana node
   - 1-zone 4GB Enterprise Search
   - Leave everything else as it is by default
   - Create the cluster and download/note down the username/password
  ![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/916ea8c4-1230-497a-bb06-cb09db57ee7c)
  ![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/7e11519d-1b73-4f19-93b2-bd7f166a72ca)


3. As a first step we need to prepare our Elastic ML nodes to create text-embedding out of content we will be indexing. We just need to load our transformer model of choice into Elastic and start it. This can be done through the [Eland Client](https://github.com/elastic/eland). We will use the [all-distillroberta-v1](https://huggingface.co/sentence-transformers/all-distilroberta-v1) ML model. To run Eland client you need docker installed. An easy way to accomplish this step without python/docker installation is via Google's Cloud Shell. Be sure the eland version you're cloning is compatible with the Elastic version you chose. If you used the latest Elastic version, there's generally no need to specify the Eland release version while cloning.
  - Enter Google Cloud's console.
  - Open the Cloud Shell editor (you can use [this link](https://console.cloud.google.com/cloudshelleditor?cloudshell=true))
  - Enter the following commands. Take a look at the last one: you have to specify your Elastic username and password previously found + the elastic endpoint (find it at Elatic admin home -> "Manage" button on your deployment --> "Copy endpoint" on the Elasticsearch line)
  
 ```bash
git clone https://github.com/elastic/eland.git

cd eland/

docker build -t elastic/eland .

docker run -it --rm elastic/eland eland_import_hub_model 
--url https://<elastic_user>:<elastic_password>@<your_elastic_endpoint>:9243/ 
--hub-model-id sentence-transformers/all-distilroberta-v1 
--start
 ```
4. After the model finishes loading into Elastic, enter your deployment and from the left menu go to "Stack Management" -> "Machine Learning". You should notice the all-distilroberta-v1 model listed and in the "started" status. If a "out of sync" warning is shown, click on it and sync. Everything should be now set. Our ML model is up-and-running. We now are able to apply our transformer model to the documents we are going to ingest.
   
5. Let's start with indexing some general retail data from the Ikea website. We will use the built-in web crawler feature, configure it this way:
   - Search for "Web Crawler" in the Elastic search bar
   - Name the new index as "search-homecraft-ikea" (watch out the "search-" prefix is already there) and go next screen.
   - Specify "https://www.ikea.com/gb/en/" in the domain URL field and click "Validate Domain". A warning should appear, saying that a robots.txt file was found. Click on it to open it in a separate browser tab and continue by clicking "Add domain"
   - For better crawling performance search the sitemap.xml filepath inside the robots.txt file of the target webserver, and add its path to the Site Maps tab.
   - To avoid crawling too many pages and stick with the english ones we can define some crawl rules. Set as follows (rule order counts!):
     ![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/1f0d52cc-2d01-4b5d-9c0e-927042ccd932)

6. Now the new empty index should be automatically created for you. Have a look at it:
   - From the left panel, in the "Enterprise Search" section, select "Content" and click on index to explore its details.
   - You can also query it from the "Management" -> "Dev Tools" UI with the following query
     ```bash
     #Query index config
     GET search-homecraft-ikea 
     #Query index content
      GET search-homecraft-ikea/_search

7. Before starting crawling we need to attach an ingest pipeline to the newly create index, so every time a new document is indexed, it will be processed by our ML model that will enrich the document with an additional field, the vector representation of its content.
   - Open the index from the Enterprise Search UI and navigate to the "Pipelines" tab
   - Click on "Copy and customize" to create a custom pipeline associated to this index
   - In the Machine Learning section click on "Add Inference Pipeline"
   - Name the pipeline as "ml-inference-title-vector"
   - Select your transformer model and go next screen
   - Select the "Source field". This is the field that the ML model will process to create vectors from, and the UI suggests the ones automatically created from the web crawler. Select the "title" field as source field, leave everything as default and go on untile pipeline is created.
  
8. Check the newly created ingest pipeline searching it from the "Stack Management" -> "Ingest pipelines" section. You are able to analyze the processors (the processing tasks) listed in the pipeline and add/remove/modify them. Note also that you can specify exception handlers.

9. Before launching the crawler we need to set the mappings for the target field where the vetors will be stored, specifying the "title-vector" field is of type "dense_vector", vector dimensions and its similarity algorithm.

```bash
POST search-homecraft-ikea/_mapping
{
  "properties": {
    "title-vector": {
      "type": "dense_vector",
      "dims": 768,
      "index": true,
      "similarity": "dot_product"
    }
  }
}
```

10. Start crawling. Go back on the index and click on the "Start Crawling" button on the top right corner of the page.

11. Let's now ingest a product catalog, to let our users search for products via hybrid search (keywords + semantics):
    - Load into Elastic this [Home Depot product catalog](https://www.kaggle.com/datasets/thedevastator/the-home-depot-products-dataset) via the "Upload File" feature (search for it in the top search bar). Click on "Import" and name the index "home-depot-product-catalog"
    - Take a look at the document structure, the fields available and their content. You will notice there are title and descriptions fields as well. We will apply the inference on the title field, so we can reuse the previously created ingest pipeline
    - From the Dev Tools console create a new empty index that will host the dense vectors called "home-depot-product-catalog-vector" and specify mappings.

```bash
PUT /home-depot-product-catalog-vector 

POST home-depot-product-catalog-vector/_mapping
{
  "properties": {
    "title-vector": {
      "type": "dense_vector",
      "dims": 768,
      "index": true,
      "similarity": "dot_product"
    }
  }
}
```

   - Re-index the product dataset through the same ingest pipeline previously created for the web-crawler. The new index will now have vectors embedded in documents in the title-vector field.

```bash
POST _reindex
{
  "source": {
    "index": "home-depot-product-catalog"
  },
  "dest": {
    "index": "home-depot-product-catalog-vector",
    "pipeline": "ml-inference-title-vector"
  }
}
```
  

  - (Note that we used these steps to show how to use reindexing and ingest pipelines via API. You can still apply the pipelines via UI as done before with the search-homecraft-ikea index)

12. Leverage the BigQuery to Elasticsearch Dataflow's [native integration](https://www.elastic.co/blog/ingest-data-directly-from-google-bigquery-into-elastic-using-google-dataflow) to move a [sample e-commerce dataset](https://console.cloud.google.com/marketplace/product/bigquery-public-data/thelook-ecommerce?project=elastic-sa) into Elastic. Take a look ad tables available in this dataset withih BigQuery explorer UI. Copy the ID of the "Order_items" table and create a new Dataflow job to move data from this BQ table to an index named "bigquery-thelook-order-items". You need to create an API key on the Elastic cluster and pass it along with Elastic cluster's cloud_id, user and pass to the job config. This new index will be used for retrieving user orders.

13. Create a small Google Cloud Compute Engine machine with default settings, with public ip address and access it via SSH. This will be used as our web-server for the front-ent application hosting our "intelligent search bar"

14. 5. Clone the [homecraft_vertex source-code repo](https://github.com/valerioarvizzigno/homecraft_vertex) on your CE machine.

```bash
git clone https://github.com/valerioarvizzigno/homecraft_vertex.git
```

15. Install requirements needed to run the app from the requirements.txt file

```bash
cd homecraft_vertex
pip install -r requirements.txt

```

16. Set up the environment variables cloud_id (the elastic CloudID - find it on the Elastic admin console), cloud_pass and cloud_user (Elastic deployments's user details) and gcp_project_id (the GCP project you're working in)

```bash
cloud_id='<replaceHereYourElasticCloudID)'
cloud_user='elastic'
cloud_pass='<replaceHereYourElasticDeploymentPassword'
gcp_project_id='<replaceHereTheGCPProjectID'

```





## Sample questions

---USE THE HOME PAGE FOR BASE DEMO---

Try queries like: 

- "List the 3 top paint primers in the product catalog, specify also the sales price for each product and product key features. Then explain in bullet points how to use a paint primer".
You can also try asking for related urls and availability --> leveraging private product catalog + public knowledge

- "could you please list the available stores in UK" --> --> it will likely use (crawled docs)

- "Which are the ways to contact customer support in the UK? What is the webpage url for customer support?" --> it will likely use crawled docs

- Please provide the social media accounts info from the company --> it will likely use crawled docs

- Please provide the full address of the Manchester store in UK --> it will likely use crawled docs

- are you offering a free parcel delivery? --> it will likely use crawled docs

- Could you please list my past orders? Please specify price for each product --> it will search into BigQuery order dataset

- List all the items I have bought in my order history in bullet points


---FOR A DEMO OF FINE-TUNED MODEL USE "HOMECRAFT FINETUNED" WEBPAGE---

Try "Anyone available at Homecraft to assist with painting my house?".
Asking this question in the fine-tuned page should suggest to go with Homecraft's network of professionals

Asking the same to the base model will likely provide a generic or "unable to help" answer.
