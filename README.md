The project will use vector embeddings and Retrieval-Augmented Generation (RAG) to generate intelligent product recommendations.

1. Create Azure OpenAI resource in azure portal and fetch the key and endpoint of the resource
2. From Microsoft foundry deploy base model for embedding-ada-002 and GPT-5

   <img width="758" height="340" alt="image" src="https://github.com/user-attachments/assets/75662be0-e871-4a67-babd-4ac11679a78f" />

3. Setup database credential
  A database scoped credential is a record in the database that contains authentication information for connecting to a resource outside the database.


-- Create a master key for the database
if not exists(select * from sys.symmetric_keys where [name] = '##MS_DatabaseMasterKey##')
begin
    create master key encryption by password = N'V3RYStr0NGP@ssw0rd!';
end
go

-- Create the database scoped credential for Azure OpenAI
if not exists(select * from sys.database_scoped_credentials where [name] = 'https://AI_ENDPOINT_SERVERNAME.openai.azure.com/')
begin
    create database scoped credential [https://AI_ENDPOINT_SERVERNAME.openai.azure.com/]
    with identity = 'HTTPEndpointHeaders', secret = '{"api-key":"YOUR_OPENAI_KEY"}';
end
go

4. Test the connectivity to Azure OpenAI and see the ability to call external REST endpoints in action. 
declare @url nvarchar(4000) = N'https://AI_ENDPOINT_SERVERNAME.openai.azure.com/openai/deployments/gpt-4/chat/completions?api-version=2024-06-01';
declare @payload nvarchar(max) = N'{"messages":[{"role":"system","content":"You are an expert joke teller."},                                   
                                {"role":"system","content":"tell me a joke about a llama walking into a bar"}]}'
declare @ret int, @response nvarchar(max);

exec @ret = sp_invoke_external_rest_endpoint
    @url = @url,
    @method = 'POST', 
    @payload = @payload,
    @credential = [https://AI_ENDPOINT_SERVERNAME.openai.azure.com/],    
    @timeout = 230,
    @response = @response output;

select json_value(@response, '$.result.choices[0].message.content') as "Amazing, Awesome, Stupendous Joke";
