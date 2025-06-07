**Q1. Running Elastic**


```python
!pip install docker
```

    Requirement already satisfied: docker in ./.venv/lib/python3.13/site-packages (7.1.0)
    Requirement already satisfied: requests>=2.26.0 in ./.venv/lib/python3.13/site-packages (from docker) (2.32.3)
    Requirement already satisfied: urllib3>=1.26.0 in ./.venv/lib/python3.13/site-packages (from docker) (2.4.0)
    Requirement already satisfied: charset-normalizer<4,>=2 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->docker) (3.4.2)
    Requirement already satisfied: idna<4,>=2.5 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->docker) (3.10)
    Requirement already satisfied: certifi>=2017.4.17 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->docker) (2025.4.26)



```python
import docker

client = docker.from_env()

container = client.containers.run(
    image="docker.elastic.co/elasticsearch/elasticsearch:8.17.6",
    name="elasticsearch",
    environment={
        "discovery.type": "single-node",
        "xpack.security.enabled": "false"
    },
    mem_limit="4g",
    ports={'9200/tcp': 9200, '9300/tcp': 9300},
    remove=True,
    detach=True,
    tty=True
)
```


```python
import requests, time

time.sleep(5)
response = requests.get("http://localhost:9200")
#print the build hash
print("The elastic search build hash is : ",response.json()['version']['build_hash'])
```

    The elastic search build hash is :  dbcbbbd0bc4924cfeb28929dc05d82d662c527b7


**Q2. Indexing the data**

Answer:

index

**Code for question 3 & 4**


```python
!pip install elasticsearch==8.18 tqdm
```

    Requirement already satisfied: elasticsearch==8.18 in ./.venv/lib/python3.13/site-packages (8.18.0)
    Requirement already satisfied: tqdm in ./.venv/lib/python3.13/site-packages (4.67.1)
    Requirement already satisfied: elastic-transport<9,>=8.15.1 in ./.venv/lib/python3.13/site-packages (from elasticsearch==8.18) (8.17.1)
    Requirement already satisfied: python-dateutil in ./.venv/lib/python3.13/site-packages (from elasticsearch==8.18) (2.9.0.post0)
    Requirement already satisfied: typing-extensions in ./.venv/lib/python3.13/site-packages (from elasticsearch==8.18) (4.14.0)
    Requirement already satisfied: urllib3<3,>=1.26.2 in ./.venv/lib/python3.13/site-packages (from elastic-transport<9,>=8.15.1->elasticsearch==8.18) (2.4.0)
    Requirement already satisfied: certifi in ./.venv/lib/python3.13/site-packages (from elastic-transport<9,>=8.15.1->elasticsearch==8.18) (2025.4.26)
    Requirement already satisfied: six>=1.5 in ./.venv/lib/python3.13/site-packages (from python-dateutil->elasticsearch==8.18) (1.17.0)



```python
#checking using python after running the above code
#import elasticsearch & tqdm (progress bar)
from elasticsearch import Elasticsearch
from tqdm.auto import tqdm

#assign the to es_client
es_client = Elasticsearch('http://localhost:9200') 


```


```python
es_client.info()
```




    ObjectApiResponse({'name': '5e6573ce33ee', 'cluster_name': 'docker-cluster', 'cluster_uuid': 'DyqpNDvEQW-kf5rBmldgZg', 'version': {'number': '8.17.6', 'build_flavor': 'default', 'build_type': 'docker', 'build_hash': 'dbcbbbd0bc4924cfeb28929dc05d82d662c527b7', 'build_date': '2025-04-30T14:07:12.231372970Z', 'build_snapshot': False, 'lucene_version': '9.12.0', 'minimum_wire_compatibility_version': '7.17.0', 'minimum_index_compatibility_version': '7.0.0'}, 'tagline': 'You Know, for Search'})




```python
#Getting Data - store it in documents
docs_url = 'https://github.com/DataTalksClub/llm-zoomcamp/blob/main/01-intro/documents.json?raw=1'
docs_response = requests.get(docs_url)
documents_raw = docs_response.json()

documents = []

for course in documents_raw:
    course_name = course['course']

    for doc in course['documents']:
        doc['course'] = course_name
        documents.append(doc)
```


```python

#setting
index_settings = {
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 0
    },
    "mappings": {
        "properties": {
            "text": {"type": "text"},
            "section": {"type": "text"},
            "question": {"type": "text"},
            "course": {"type": "keyword"} 
        }
    }
}

index_name = "course-questions"

#creating the index with the settings
es_client.indices.create(index=index_name, body=index_settings)

#indexing the documents
for doc in tqdm(documents):
    es_client.index(
        index=index_name,
        document=doc
    )
```

    100%|██████████| 948/948 [00:04<00:00, 231.08it/s]


**Q3. Searching**


```python

#defining the search query
query= "How do execute a command on a Kubernetes pod?"
```


```python

#search query setting
search_query = {
        "size": 5,
        "query": {
            "bool": {
                "must": {
                    "multi_match": {
                        "query": query,
                        "fields": ["question^4", "text"],
                        "type": "best_fields"
                    }
                }
            }
        }
    }

#run the search
response = es_client.search(index=index_name, body=search_query)
result_docs = []

#find out what we need
for hit in response['hits']['hits']:
    result_docs.append(hit['_score'])

#find the top ranking result
print("Top ranking result score:", max(result_docs))
```

    Top ranking result score: 44.50556


if es_client.indices.exists(index='course-questions'):
    es_client.indices.delete(index='course-questions')
    print("Index successfully deleted.")
else:
    print("Index doesn't exist.")

**Q4. Filtering**


```python
#defining the search query
query= "How do copy a file to a Docker container?"
```


```python

#search query setting
search_query = {
        "size": 3,
        "query": {
            "bool": {
                "must": {
                    "multi_match": {
                        "query": query,
                        "fields": ["question^4", "text"],
                        "type": "best_fields"
                    }
                },
                "filter": {
                    "term": {
                        "course": "machine-learning-zoomcamp"
                    }
                }
            }
        }
    }

#run the search
response = es_client.search(index=index_name, body=search_query)
result_docs = []

#find out what we need
for hit in response['hits']['hits']:
    result_docs.append(hit['_source'])

i=1
for doc in result_docs:
    print(f"Question {i}: {doc['question']}")
    i += 1


```

    Question 1: How do I debug a docker container?
    Question 2: How do I copy files from my local machine to docker container?
    Question 3: How do I copy files from a different folder into docker container’s working directory?


**Q5. Building a prompt**


```python
query = "How do I execute a command in a running docker container?"
```


```python
def build_prompt(query, search_results):
    prompt_template = """
You're a course teaching assistant. Answer the QUESTION based on the CONTEXT from the FAQ database.
Use only the facts from the CONTEXT when answering the QUESTION.

QUESTION: {question}

CONTEXT:
{context}
""".strip()

    context = ""
    
    for doc in search_results:
        context = context + f"Q:{doc['question']}\nA:{doc['text']}\n\n"
    
    context = context.strip()
    
    prompt = prompt_template.format(question=query, context=context).strip()
    return prompt
```


```python
prompt_result = build_prompt(query, result_docs)
prompt_result

```




    'You\'re a course teaching assistant. Answer the QUESTION based on the CONTEXT from the FAQ database.\nUse only the facts from the CONTEXT when answering the QUESTION.\n\nQUESTION: How do I execute a command in a running docker container?\n\nCONTEXT:\nQ:How do I debug a docker container?\nA:Launch the container image in interactive mode and overriding the entrypoint, so that it starts a bash command.\ndocker run -it --entrypoint bash <image>\nIf the container is already running, execute a command in the specific container:\ndocker ps (find the container-id)\ndocker exec -it <container-id> bash\n(Marcos MJD)\n\nQ:How do I copy files from my local machine to docker container?\nA:You can copy files from your local machine into a Docker container using the docker cp command. Here\'s how to do it:\nTo copy a file or directory from your local machine into a running Docker container, you can use the `docker cp command`. The basic syntax is as follows:\ndocker cp /path/to/local/file_or_directory container_id:/path/in/container\nHrithik Kumar Advani\n\nQ:How do I copy files from a different folder into docker container’s working directory?\nA:You can copy files from your local machine into a Docker container using the docker cp command. Here\'s how to do it:\nIn the Dockerfile, you can provide the folder containing the files that you want to copy over. The basic syntax is as follows:\nCOPY ["src/predict.py", "models/xgb_model.bin", "./"]\t\t\t\t\t\t\t\t\t\t\tGopakumar Gopinathan'




```python
len(prompt_result)
```




    1456




```python
context = ""
for doc in result_docs:
    print(f"Q: {doc['question']}\nA: {doc['text']}\n\n")
    context = context + f"Q: {doc['question']}\nA: {doc['text']}\n\n"

context = context.strip()
context
```

    Q: How do I debug a docker container?
    A: Launch the container image in interactive mode and overriding the entrypoint, so that it starts a bash command.
    docker run -it --entrypoint bash <image>
    If the container is already running, execute a command in the specific container:
    docker ps (find the container-id)
    docker exec -it <container-id> bash
    (Marcos MJD)
    
    
    Q: How do I copy files from my local machine to docker container?
    A: You can copy files from your local machine into a Docker container using the docker cp command. Here's how to do it:
    To copy a file or directory from your local machine into a running Docker container, you can use the `docker cp command`. The basic syntax is as follows:
    docker cp /path/to/local/file_or_directory container_id:/path/in/container
    Hrithik Kumar Advani
    
    
    Q: How do I copy files from a different folder into docker container’s working directory?
    A: You can copy files from your local machine into a Docker container using the docker cp command. Here's how to do it:
    In the Dockerfile, you can provide the folder containing the files that you want to copy over. The basic syntax is as follows:
    COPY ["src/predict.py", "models/xgb_model.bin", "./"]											Gopakumar Gopinathan
    
    





    'Q: How do I debug a docker container?\nA: Launch the container image in interactive mode and overriding the entrypoint, so that it starts a bash command.\ndocker run -it --entrypoint bash <image>\nIf the container is already running, execute a command in the specific container:\ndocker ps (find the container-id)\ndocker exec -it <container-id> bash\n(Marcos MJD)\n\nQ: How do I copy files from my local machine to docker container?\nA: You can copy files from your local machine into a Docker container using the docker cp command. Here\'s how to do it:\nTo copy a file or directory from your local machine into a running Docker container, you can use the `docker cp command`. The basic syntax is as follows:\ndocker cp /path/to/local/file_or_directory container_id:/path/in/container\nHrithik Kumar Advani\n\nQ: How do I copy files from a different folder into docker container’s working directory?\nA: You can copy files from your local machine into a Docker container using the docker cp command. Here\'s how to do it:\nIn the Dockerfile, you can provide the folder containing the files that you want to copy over. The basic syntax is as follows:\nCOPY ["src/predict.py", "models/xgb_model.bin", "./"]\t\t\t\t\t\t\t\t\t\t\tGopakumar Gopinathan'



**Q6. Tokens**


```python
!pip install tiktoken
```

    Collecting tiktoken
      Using cached tiktoken-0.9.0-cp313-cp313-macosx_11_0_arm64.whl.metadata (6.7 kB)
    Collecting regex>=2022.1.18 (from tiktoken)
      Using cached regex-2024.11.6-cp313-cp313-macosx_11_0_arm64.whl.metadata (40 kB)
    Requirement already satisfied: requests>=2.26.0 in ./.venv/lib/python3.13/site-packages (from tiktoken) (2.32.3)
    Requirement already satisfied: charset-normalizer<4,>=2 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->tiktoken) (3.4.2)
    Requirement already satisfied: idna<4,>=2.5 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->tiktoken) (3.10)
    Requirement already satisfied: urllib3<3,>=1.21.1 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->tiktoken) (2.4.0)
    Requirement already satisfied: certifi>=2017.4.17 in ./.venv/lib/python3.13/site-packages (from requests>=2.26.0->tiktoken) (2025.4.26)
    Using cached tiktoken-0.9.0-cp313-cp313-macosx_11_0_arm64.whl (1.0 MB)
    Using cached regex-2024.11.6-cp313-cp313-macosx_11_0_arm64.whl (284 kB)
    Installing collected packages: regex, tiktoken
    [2K   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m2/2[0m [tiktoken]
    [1A[2KSuccessfully installed regex-2024.11.6 tiktoken-0.9.0



```python
import tiktoken
```


```python
def num_tokens_from_string(string: str, encoding_name: str) -> int:
    """Returns the number of tokens in a text string."""
    encoding = tiktoken.encoding_for_model(encoding_name)
    num_tokens = len(encoding.encode(string))
    return num_tokens
```


```python
#number of tokens in the prompt
prompt_token = num_tokens_from_string(prompt_result, "gpt-4o")
print("Number of tokens in the prompt:", prompt_token)
```

    Number of tokens in the prompt: 322


**Q7 Bonus: generating the answer (ungraded)**


```python
pip install openai
```

    Collecting openai
      Downloading openai-1.84.0-py3-none-any.whl.metadata (25 kB)
    Collecting anyio<5,>=3.5.0 (from openai)
      Using cached anyio-4.9.0-py3-none-any.whl.metadata (4.7 kB)
    Collecting distro<2,>=1.7.0 (from openai)
      Using cached distro-1.9.0-py3-none-any.whl.metadata (6.8 kB)
    Collecting httpx<1,>=0.23.0 (from openai)
      Using cached httpx-0.28.1-py3-none-any.whl.metadata (7.1 kB)
    Collecting jiter<1,>=0.4.0 (from openai)
      Using cached jiter-0.10.0-cp313-cp313-macosx_11_0_arm64.whl.metadata (5.2 kB)
    Collecting pydantic<3,>=1.9.0 (from openai)
      Using cached pydantic-2.11.5-py3-none-any.whl.metadata (67 kB)
    Collecting sniffio (from openai)
      Using cached sniffio-1.3.1-py3-none-any.whl.metadata (3.9 kB)
    Requirement already satisfied: tqdm>4 in ./.venv/lib/python3.13/site-packages (from openai) (4.67.1)
    Requirement already satisfied: typing-extensions<5,>=4.11 in ./.venv/lib/python3.13/site-packages (from openai) (4.14.0)
    Requirement already satisfied: idna>=2.8 in ./.venv/lib/python3.13/site-packages (from anyio<5,>=3.5.0->openai) (3.10)
    Requirement already satisfied: certifi in ./.venv/lib/python3.13/site-packages (from httpx<1,>=0.23.0->openai) (2025.4.26)
    Collecting httpcore==1.* (from httpx<1,>=0.23.0->openai)
      Using cached httpcore-1.0.9-py3-none-any.whl.metadata (21 kB)
    Collecting h11>=0.16 (from httpcore==1.*->httpx<1,>=0.23.0->openai)
      Using cached h11-0.16.0-py3-none-any.whl.metadata (8.3 kB)
    Collecting annotated-types>=0.6.0 (from pydantic<3,>=1.9.0->openai)
      Using cached annotated_types-0.7.0-py3-none-any.whl.metadata (15 kB)
    Collecting pydantic-core==2.33.2 (from pydantic<3,>=1.9.0->openai)
      Using cached pydantic_core-2.33.2-cp313-cp313-macosx_11_0_arm64.whl.metadata (6.8 kB)
    Collecting typing-inspection>=0.4.0 (from pydantic<3,>=1.9.0->openai)
      Using cached typing_inspection-0.4.1-py3-none-any.whl.metadata (2.6 kB)
    Downloading openai-1.84.0-py3-none-any.whl (725 kB)
    [2K   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m725.5/725.5 kB[0m [31m7.8 MB/s[0m eta [36m0:00:00[0m
    [?25hUsing cached anyio-4.9.0-py3-none-any.whl (100 kB)
    Using cached distro-1.9.0-py3-none-any.whl (20 kB)
    Using cached httpx-0.28.1-py3-none-any.whl (73 kB)
    Using cached httpcore-1.0.9-py3-none-any.whl (78 kB)
    Using cached jiter-0.10.0-cp313-cp313-macosx_11_0_arm64.whl (318 kB)
    Using cached pydantic-2.11.5-py3-none-any.whl (444 kB)
    Using cached pydantic_core-2.33.2-cp313-cp313-macosx_11_0_arm64.whl (1.8 MB)
    Using cached annotated_types-0.7.0-py3-none-any.whl (13 kB)
    Using cached h11-0.16.0-py3-none-any.whl (37 kB)
    Using cached sniffio-1.3.1-py3-none-any.whl (10 kB)
    Using cached typing_inspection-0.4.1-py3-none-any.whl (14 kB)
    Installing collected packages: typing-inspection, sniffio, pydantic-core, jiter, h11, distro, annotated-types, pydantic, httpcore, anyio, httpx, openai
    [2K   [90m━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━[0m [32m12/12[0m [openai]11/12[0m [openai]c]
    [1A[2KSuccessfully installed annotated-types-0.7.0 anyio-4.9.0 distro-1.9.0 h11-0.16.0 httpcore-1.0.9 httpx-0.28.1 jiter-0.10.0 openai-1.84.0 pydantic-2.11.5 pydantic-core-2.33.2 sniffio-1.3.1 typing-inspection-0.4.1
    Note: you may need to restart the kernel to use updated packages.



```python
from openai import OpenAI
client = OpenAI()
```


```python
def llm(prompt):
    response = client.chat.completions.create(
        model='gpt-4o',
        messages=[{"role": "user", "content": prompt}]
    )
    
    return response.choices[0].message.content
```


```python
#RUN AT YOUR COST
answer = llm(prompt_result)
answer
```




    "To execute a command in a running Docker container, use the following command:\n\n```bash\ndocker exec -it <container-id> bash\n```\n\nFirst, you'll need to find the container ID of the running container by using the `docker ps` command. Once you have the container ID, you can replace `<container-id>` in the command above with the actual ID to execute the command inside the container."




```python
#number of tokens in the answer
answer_token = num_tokens_from_string(answer, "gpt-4o")
print("Number of tokens in the answer:", answer_token)
```

    Number of tokens in the answer: 82


**Q8 Bonus: calculating the costs (ungraded)**

On the 6th June 2025
GPT 4.0 Text Cost

$5.00 / 1M input token -> 0.00005c per token
$20.00 / 1M output token -> 0.0002c per token


```python
#prompt token cost
prompt_cost = prompt_token * 0.05 / 1000
#answer token cost
answer_cost = answer_token * 0.2 / 1000

print(f"Prompt cost: ${prompt_cost:.6f}")
print(f"Answer cost: ${answer_cost:.6f}")
```

    Prompt cost: $0.016100
    Answer cost: $0.016400



```python

```
