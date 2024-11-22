## Build RAG based generative AI applications in AWS using Amazon FSx for NetApp ONTAP with Amazon Bedrock

1. Use Amazon FSx for NetApp ONTAP with Amazon Bedrock to build Retrieval Augmented Generation (RAG) based generative AI applications by bringing company-specific, unstructured user file data from FSx for NetApp ONTAP for your RAG applications.
2. Leverage Windows and Linux based file ownership-based access control operations to provide a permissions-based RAG experience for your users.

### Solution Overview

The solution uses an FSx for ONTAP filesystem as the source of unstructured data and continuously populates an Amazon OpenSearch Serverless (AOSS) vector database with the user’s existing files and folders and associated metadata.  This enables a RAG scenario with Amazon Bedrock by enriching the generative AI prompt using Bedrock APIs with your company-specific data retrieved from the OpenSearch Serverless vector database. 

The solution also leverages FSx for ONTAP to allow users to extend their current data security and access mechanisms to augment model responses from Bedrock. It uses FSx for ONTAP as the source of associated metadata, specifically the user’s security access control list (ACL) configurations attached to their files and folders and populates that metadata into AOSS. By combining access control operations with file events that notify the RAG application of new and changed data on the file system, we demonstrate how FSx for NetApp ONTAP enables Bedrock to only use embeddings from authorized files for the specific users that connect to our generative AI application.

The solution provisions a multi-AZ deployment of the FSx for ONTAP filesystem with a storage virtual machine (SVM) joined to an AWS Managed Microsoft AD domain. An Amazon OpenSearch Serverless(AOSS) vector search collection provides scalable and high performing similarity search capability. We use an Amazon EC2 Windows server as an SMB/CIFS client to the FSx for ONTAP volume and configure data sharing and ACLs for the SMB shares in the volume. We use this data and ACLs to test permissions-based access to the embeddings in a RAG scenario with Bedrock.

The Embeddings container component of our solution that is deployed on an Amazon EC2 Linux server and mounted as an NFS client on the FSx for ONTAP volume, periodically migrates existing files and folders along with their security access control list (ACL) configurations to AOSS. It populates an index in the AOSS vector search collection with this company specific data (and associated metadata and ACLs) from the NFS share on the FSx for ONTAP file system.

The solution implements a RAG Retrieval Lambda function that enables a RAG scenario with Amazon Bedrock by enriching the Generative AI prompt using Bedrock APIs with your company-specific data and associated metadata (including ACLs) retrieved from the Amazon OpenSearch Serverless index that was populated by the Embeddings container component described above. The RAG Retrieval Lambda function also stores conversation history for the user interaction in an Amazon DynamoDB table.

The user interacts with the solution by submitting a natural language prompt either via a chatbot application or directly via the Amazon API Gateway API interface. The chatbot application container is built using streamlit and fronted by an AWS Application Load Balancer(ALB). When a user submits a natural language prompt to the chatbot UI via the ALB, the chatbot container interacts with API Gateway interface that then invokes the RAG Retrieval Lambda function to fetch the response for the user. The user can also directly submit prompt requests to the Amazon API Gateway API and obtain a response. We currently demonstrate permissions-based access to the RAG documents by explicitly retrieving the SID of a user and then using that SID in the chatbot or API Gateway request where the Retrieval lambda then matches the SID to the Windows ACLs configured for the document. In general, as a future enhancement, you may want to authenticate the user against an Identity Provider and then match the user against the permissions configured for the documents.

The following diagram illustrates the end-to-end flow for our solution. We start by configuring data sharing and ACLs with FSx for NetApp ONTAP and then these are periodically scanned by the Embeddings container. The Embeddings container splits the documents into chunks and uses the Amazon Titan Embeddings model to create vector embeddings from these chunks. It then stores these vector embeddings with associated metadata in our AOSS vector database by populating an index in a vector collection in AOSS.

![Embedding Flow](/images/flow-arch.png)

Here’s the overall reference architecture diagram that illustrates the various components of this solution working together

![Reference Architecture](/images/solution-arch.png)

### Prerequisites

1. Ensure you have [model access in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-access.html) for both the Anthropic Claude v3 and Titan Text Embedding models available on Amazon Bedrock.
2. Install [AWS CLI](https://aws.amazon.com/cli)
3. Install [Docker](https://docs.docker.com/engine/install/)
4. [Install Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)

### Setup

Cloning the repository and using the Terraform template will provision all the components with their required configurations as described in the solution overview section.

1. Clone the repository for this solution:
```
sudo yum install -y unzip
git clone https://github.com/aws-samples/genai-bedrock-fsxontap.git
cd genai-bedrock-fsxontap/terraform
```
2. From the _terraform_ folder, deploy the entire solution using terraform:
```
terraform init
terraform apply -auto-approve
```
This process can take 15–20 minutes to complete.

### Test and Validate

#### Load data and set permissions

Use the _ad_host_ instance to share sample data and set user permissions that will then be seamlessly used to populate the index on AOSS by the solution’s Embedding container component. Perform the following steps to mount your Amazon FSx for ONTAP storage virtual machine data volume as a network drive, upload data to this shared network drive and set permissions based on Windows ACLs:

1. Navigate to Systems Manager Fleet Manager on your AWS console, locate the  _ad_host_ instance and [follow instructions here](https://docs.aws.amazon.com/systems-manager/latest/userguide/fleet-rdp.html#fleet-rdp-connect-to-node) to login with Remote Desktop. Use the domain admin user _bedrock-01\\Admin_ and your _AD host password_ obtained from the Terraform output
2. Mount FSxN data volume as a network drive. Navigate to FSx on your AWS console, click on **Volumes**, select the *bedrockrag* volume and obtain values for the SVM ID and Volume name parameters. Under **This PC** right click **Network** and then **Map Network drive.** Choose drive letter and use the FSxN share path for the mount (\\\\&lt;SVM ID&gt;.brsvm.bedrock-01.com\\c$\\&lt;Volume name&gt;)
3. Upload the [Bedrock user guide](https://docs.aws.amazon.com/pdfs/bedrock/latest/userguide/bedrock-ug.pdf) to the shared network drive and set permissions to the Admin user only (ensure that you **Disable inheritance** under **Advanced settings**)
4. Upload the [FSx ONTAP user guide](https://docs.aws.amazon.com/pdfs/fsx/latest/ONTAPGuide/ONTAPGuide.pdf#getting-started) to the shared drive and ensure permissions are set to Everyone
5. On the _ad_host_ server open the command prompt and type the following command to obtain the SID for the Admin user:
```
wmic useraccount where name='Admin' get sid
```

#### Start periodic migration of FSxN data to AOSS

Mount and run the Embeddings container to periodically populate an index in the AOSS vector search collection with existing files and folders along with their security access control list (ACL) configurations from the FSx for ONTAP file system.

1. Navigate to Systems Manager Fleet Manager on your AWS console, locate the  _embedding_host_ instance, click the **Node actions** button and select **Start terminal session** to login to the _embedding_host_

> Optional: If Embedding container fails because of FSxN storage mount makse sure you mount and start the Embeddings Container. 
>
> 1. Navigate to FSx ONTAP on your AWS console and click on **Storage Virtual Machines** and obtain values for the SVM ID and File System ID parameters. 
> 2. Navigate to Amazon Elastic Container Registry on your AWS console and click on **Repositories** and obtain the value of the URI parameter for the fsxnragembed image name.
> 3. Run the following:  
>
>```
>sudo mount -t nfs <SVM ID>.<File system ID>:/ragdb /tmp/db
>sudo docker run -d -v /tmp/data:/opt/netapp/ai/data -v /tmp/db:/opt/netapp/ai/db -e ENV_REGION=<AWS_REGION> -e ENV_OPEN_SEARCH_SERVERLESS_COLLECTION_NAME=fsxnragvector <URI for fsxnragembed image>
>```
#### Test permissions-based RAG scenario with Bedrock and FSx for ONTAP

##### Use the Chatbot

Obtain the _lb-dns-name_ URL from the output of your Terraform template and access it via your web browser.

For the prompt query, ask any general question on the FSxN user guide that is available for access to everyone. You will see a response in the chat window as well as the source attribution used by the model for the response.

Now ask a question about the Bedrock user guide that has access restricted to the Admin user. You can see the model will not return an answer related to this query. Use the Admin SID on the User (SID) filter search in the chat UI and ask the same question again in the prompt. You will now see a response in the chat window as well as the source attribution used by the model for the response.

##### Query using API Gateway

You can also query the model directly using API Gateway. Obtain the *api-invoke-url* parameter from the output of your Terraform template

1. Here’s the curl request you can use for invoking API Gateway with *Everyone* access for a query related to the FSxN user guide. Note that the value of the *metadata* parameter is set to *NA* to indicate *Everyone* access. 
```
curl -v '<api-invoke-url>/bedrock_rag_retreival' -X POST -H 'content-type: application/json' -d '{"session_id": "1","prompt": "What is an FSxN ONTAP filesystem?", "bedrock_model_id": "anthropic.claude-3-sonnet-20240229-v1:0", "model_kwargs": {"temperature": 1.0, "top_p": 1.0, "top_k": 500}, "metadata": "NA", "memory_window": 10}'
```
2. Here’s the curl request you can use for invoking API Gateway with *Admin* access for a query related to the Bedrock user guide. Note that the value of the *metadata* parameter is set to the *SID* of the Admin user.
```
curl -v '<api-invoke-url>/bedrock_rag_retreival' -X POST -H 'content-type: application/json' -d '{"session_id": "1","prompt": "what is bedrock?", "bedrock_model_id": "anthropic.claude-3-sonnet-20240229-v1:0", "model_kwargs": {"temperature": 1.0, "top_p": 1.0, "top_k": 500}, "metadata": "S-1-5-21-4037439088-1296877785-2872080499-1112", "memory_window": 10}'
```

### Clean up

To avoid recurring charges, and to clean up your account after trying the solution outlined in this post, perform the following steps:

1. From the _terraform_ folder, delete the Terraform template for the solution:
```
terraform apply --destroy
```


##### FAQs

FAQs for STG317 "Build RAG based GenAI applications on AWS using Amazon FSx for NetApp ONTAP"

o	Q: What is RAG in the context of GenAI applications?
o	A: RAG stands for Retrieval-Augmented Generation. It's an approach that combines information retrieval with generative AI to produce more accurate and contextually relevant responses.

o	Q: What is the primary data source for this RAG-based GenAI solution?
o	A: The primary data source is an Amazon FSx for NetApp ONTAP (FSx for ONTAP) filesystem.

o	Q: What is Amazon FSx for NetApp ONTAP?
o	A: It's a fully managed file storage service that combines AWS's scalability with NetApp's ONTAP file system, providing high-performance, feature-rich file storage in the cloud.

o	Q: Why use FSx for NetApp ONTAP as the data source?
o	A: Amazon FSx for NetApp ONTAP provides scalable, high-performance storage designed for large datasets and I/O-intensive GenAI workloads, offering efficient data retrieval, management, encryption, access controls, dynamic scaling, high concurrency support, efficient metadata handling, and built-in deduplication and data compression. It is compatible with container orchestration platforms and incorporates AWS security features, ensuring data protection throughout the GenAI pipeline.

o	Q: What provides the vector search capability in this solution and why?
o	A: An Amazon OpenSearch Serverless (AOSS) is the vector database used because it provides scalable and high-performing similarity searches.

o	Q: Can other vector databases be used in this solution?
o	A: Yes. In addition to Amazon OpenSearch Serverless, other vector databases that can be used for RAG applications. These databases offer various features like high-performance similarity search, scalability, and integration with different machine learning frameworks, allowing developers to choose the one that best fits their specific requirements and infrastructure setup.

o	Q: How is the vector database populated in this solution?
o	A: An Embeddings container component periodically migrates existing files, folders, and their ACLs from the FSx for ONTAP filesystem to an Amazon OpenSearch Serverless (AOSS) vector database.

o	Q: How does the solution handle data security and access control?
o	A: It leverages the existing security and access mechanisms from FSx for ONTAP, including user ACLs, and extends them to the RAG application.

o	Q: What ensures that Bedrock only uses authorized files for each user?
o	A: The solution combines access control operations with file events, allowing the RAG application to use only embeddings from files the user is authorized to access.

o	Q: Can this solution use any FSx for ONTAP deployment configuration?
o	A: Yes, it can use either multi-AZ deployment or single AZ deployment with a storage virtual machine (SVM) joined to an AWS Managed Microsoft AD domain.

o	Q: How are Windows ACLs configured and tested in the solution?
o	A: An Amazon EC2 Windows server is used as an SMB/CIFS client to configure data sharing and ACLs for the SMB shares in the FSx for ONTAP volume.

o	Q: What component is responsible for generating and storing embeddings?
o	A: The Embeddings container, deployed on an Amazon EC2 Linux server and mounted as an NFS client on the FSx for ONTAP volume, generates embeddings using the Amazon Titan Embeddings model.

o	Q: How does the RAG Retrieval process work in this solution?
o	A: A RAG Retrieval Lambda function enriches the GenAI prompt with company-specific data retrieved from the AOSS index, using Bedrock APIs.

o	Q: Where is the conversation history stored?
o	A: The conversation history is stored in an Amazon DynamoDB table.

o	Q: Why use DynamoDB for storing conversation history?
o	A: Amazon DynamoDB is a serverless, NoSQL, fully managed database with single-digit millisecond performance at any scale. It is an excellent choice for storing conversation history in RAG applications due to its scalability, low-latency performance, and flexible NoSQL schema that can accommodate varying conversation structures. It offers seamless integration with other AWS services, robust security features, and cost-effective pricing, making it well-suited for managing both real-time interactions and long-term storage of conversation data in a production environment.  

o	Q: How can users interact with the solution?
o	A: Users can interact either through a chatbot application or directly via an Amazon API Gateway API interface.

o	Q: What technology is used for the chatbot application?
o	A: The chatbot application is built using Langchain.

o	Q: Why use Langchain and Streamlit for the chatbot application?
o	A: LangChain and Streamlit are commonly used together to create user-friendly LLM-powered applications. Langchain is a framework for developing applications powered by language models (LLMs). It primarily focusses on the backend logic and LLM integration. It helps in combining LLMs with other sources of computation or knowledge. 

o	Q: What technology is used for the chatbot User Interface?
o	A: The chatbot UI is powered by Streamlit and is fronted by an AWS Application Load Balancer (ALB).

o	Q: Why use Streamlit for the chatbot User Interface?
o	A: LangChain and Streamlit are commonly used together to create user-friendly LLM-powered applications. Streamlit is chosen for the chatbot user interface due to its simplicity in creating interactive web applications with Python and its rapid prototyping capabilities. It allows developers to quickly build and deploy user-friendly interfaces for AI applications, including chatbots, with minimal front-end development effort, making it an efficient choice for showcasing and iterating on RAG-based conversational AI systems.

o	Q: Why use the AWS Application Load Balancer (ALB) to front the chatbot?
o	A: The AWS Application Load Balancer (ALB) is used to front the chatbot for two primary reasons: it provides high availability and scalability by distributing incoming traffic across multiple instances of the chatbot application, and it offers advanced routing capabilities and built-in security features. This setup ensures the chatbot can handle varying levels of user traffic efficiently while maintaining performance and reliability.

o	Q: What type of built in security features?
o	A: AWS Application Load Balancer (ALB) offers several built-in security features:
o	 SSL/TLS termination for encrypted traffic.
o	Integration with AWS WAF for protection against web exploits
o	User authentication through integration with identity providers.
o	Support for security groups to control inbound and outbound traffic.
o	Access logging for security analysis and auditing

o	Q: How is permissions-based access to RAG documents currently demonstrated?
o	A: It's demonstrated by explicitly retrieving the SID of a user and using that SID in the chatbot or API Gateway request, which is then matched to the Windows ACLs configured for the document.

o	Q: Can future enhancements be used for user authentication?
o	A: Yes, Users can authenticate against an Identity Provider which can match the user’s credentials against the permissions configured for the documents.

o	Q: How are documents processed for embedding?
o	A: The Embeddings container splits the documents into chunks before creating vector embeddings using the Amazon Titan Embeddings model.

o	Q: Can other chunking methods be used for the embeddings?
o	A: Yes.

o	Q: How does the solution ensure data consistency between FSx for ONTAP and the AOSS vector database?
o	A: The Embeddings container periodically scans and updates the AOSS index with any changes in the FSx for ONTAP filesystem.

o	Q: Can this solution handle real-time updates to the vector database from the data source?
o	A: While the solution periodically updates the vector database, real-time updates can be achieved with additional implementation.

o	Q: How does the solution scale to handle large volumes of data?
o	A: It leverages the scalability of FSx for ONTAP for storage and AOSS for vector search, both of which are designed to handle large-scale operations.

o	Q: What role does Amazon Bedrock play in this solution?
o	A: Amazon Bedrock is a fully managed service that makes high-performing foundation models (FMs) from leading AI companies and Amazon available for your use through a unified API. It provides the APIs for interacting with the generative AI model and the embedding model (Amazon Titan).

o	Q: How does the solution handle multi-user concurrent access?
o	A: The solution uses ALB for the chatbot interface and API Gateway for direct API access, both of which support concurrent users.

o	Q: What types of files can be processed by this solution?
o	A: The solution can handle various unstructured data types supported by FSx for ONTAP, though specific file type handling may depend on the Embeddings container implementation.

o	Q: Can users search across multiple FSx for ONTAP filesystems?
o	A: This capability could potentially be extended to support multiple filesystems.

o	Q: Can other AI models be used in this solution? 
o	A: By using Amazon Bedrock, updates to underlying AI models can be achieved.

o	Q: Is there a limit to the size of documents that can be processed?
o	A: There may be practical limits based on the Embeddings container and AOSS configurations.


