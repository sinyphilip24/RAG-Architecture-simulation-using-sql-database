The project will use vector embeddings and Retrieval-Augmented Generation (RAG) to generate intelligent product recommendations.

1. Create Azure OpenAI resource in azure portal and fetch the key and endpoint of the resource
2. From Microsoft foundry deploy base model for embedding-ada-002 and GPT-5

   <img width="758" height="340" alt="image" src="https://github.com/user-attachments/assets/75662be0-e871-4a67-babd-4ac11679a78f" />

3. Setup database credential
  A database scoped credential is a record in the database that contains authentication information for connecting to a resource outside the database.


-- Create a master key for the database
if not exists(select * from sys.symmetric_keys where [name] = '##MS_DatabaseMasterKey##')
begin
    create master key encryption by password = ';
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

<img width="1205" height="508" alt="image" src="https://github.com/user-attachments/assets/182a37e0-b41e-4886-a45b-29ee3e4e13a4" />



4. Create embeddings for relational data
An embedding is a special format of data representation that machine learning models and algorithms can easily use. The embedding is an information dense representation of the semantic meaning of a piece of text. Each embedding is a vector of floating-point numbers. Vector embeddings can help with semantic search by capturing the semantic similarity between terms. For example, "cat" and "kitty" have similar meanings, even though they are spelled differently. Embeddings created and stored in the Azure SQL Database in Microsoft Fabric during this lab will power a vector similarity search in a chat app you will build.

Script to call an Azure openAI embeddings endpoint. 
declare @url nvarchar(4000) = N'https://AI_ENDPOINT_SERVERNAME.openai.azure.com/openai/deployments/text-embedding-ada-002/embeddings?api-version=2024-06-01';
declare @message nvarchar(max) = 'Hello World!';
declare @payload nvarchar(max) = N'{"input": "' + @message + '"}';

declare @ret int, @response nvarchar(max);

exec @ret = sp_invoke_external_rest_endpoint 
    @url = @url,
    @method = 'POST',
    @payload = @payload,
    @credential = [https://AI_ENDPOINT_SERVERNAME.openai.azure.com/],
    @timeout = 230,
    @response = @response output;

select json_query(@response, '$.result.data[0].embedding') as "JSON Vector Array";

<img width="1291" height="638" alt="image" src="https://github.com/user-attachments/assets/a2156ab0-24cf-4e44-9150-f973790dbc44" />


alter table [SalesLT].[Product]
add  embeddings VECTOR(1536), chunk nvarchar(2000);

Next, we are going to use the External REST Endpoint Invocation procedure (sp_invoke_external_rest_endpoint) to create a stored procedure that will create embeddings for text we supply as an input. Copy and paste the following code into a blank query editor in Microsoft Fabric:

create or alter procedure dbo.create_embeddings
(
    @input_text nvarchar(max),
    @embedding vector(1536) output
)
AS
BEGIN
declare @url varchar(max) = 'https://AI_ENDPOINT_SERVERNAME.openai.azure.com/openai/deployments/text-embedding-ada-002/embeddings?api-version=2024-06-01';
declare @payload nvarchar(max) = json_object('input': @input_text);
declare @response nvarchar(max);
declare @retval int;

-- Call to Azure OpenAI to get the embedding of the search text
begin try
    exec @retval = sp_invoke_external_rest_endpoint
        @url = @url,
        @method = 'POST',
        @credential = [https://AI_ENDPOINT_SERVERNAME.openai.azure.com/],
        @payload = @payload,
        @response = @response output;
end try
begin catch
    select 
        'SQL' as error_source, 
        error_number() as error_code,
        error_message() as error_message
    return;
end catch
if (@retval != 0) begin
    select 
        'OPENAI' as error_source, 
        json_value(@response, '$.result.error.code') as error_code,
        json_value(@response, '$.result.error.message') as error_message,
        @response as error_response
    return;
end
-- Parse the embedding returned by Azure OpenAI
declare @json_embedding nvarchar(max) = json_query(@response, '$.result.data[0].embedding');

-- Convert the JSON array to a vector and set return parameter
set @embedding = CAST(@json_embedding AS VECTOR(1536));
END;

Again, open a new query editor. Run the following T-SQL in a blank query editor in Microsoft Fabric to create embeddings for all products in the Products table:

SET NOCOUNT ON
DROP TABLE IF EXISTS #MYTEMP 
DECLARE @ProductID int
declare @text nvarchar(max);
declare @vector vector(1536);
SELECT * INTO #MYTEMP FROM [SalesLT].Product
SELECT @ProductID = ProductID FROM #MYTEMP
SELECT TOP(1) @ProductID = ProductID FROM #MYTEMP
WHILE @@ROWCOUNT <> 0
BEGIN
    set @text = (SELECT p.Name + ' '+ ISNULL(p.Color,'No Color') + ' '+  c.Name + ' '+  m.Name + ' '+  ISNULL(d.Description,'')
                    FROM 
                    [SalesLT].[ProductCategory] c,
                    [SalesLT].[ProductModel] m,
                    [SalesLT].[Product] p
                    LEFT OUTER JOIN
                    [SalesLT].[vProductAndDescription] d
                    on p.ProductID = d.ProductID
                    and d.Culture = 'en'
                    where p.ProductCategoryID = c.ProductCategoryID
                    and p.ProductModelID = m.ProductModelID
                    and p.ProductID = @ProductID);
    exec dbo.create_embeddings @text, @vector output;
    update [SalesLT].[Product] set [embeddings] = @vector, [chunk] = @text where ProductID = @ProductID;
    DELETE FROM #MYTEMP WHERE ProductID = @ProductID
    SELECT TOP(1) @ProductID = ProductID FROM #MYTEMP
END


To ensure all the embeddings were created, run the following code and you should get 0 for the result.
select count(*) from SalesLT.Product where embeddings is null

<img width="1314" height="812" alt="image" src="https://github.com/user-attachments/assets/009d1700-3586-4d49-a319-31d403705063" />


<img width="1313" height="657" alt="image" src="https://github.com/user-attachments/assets/dd832b6a-74bc-4637-b70c-4acaf1faa1c5" />

Task-5: Vector Similarity Searching
Vector similarity searching is a technique used to find and retrieve data points that are similar to a given query, based on their vector representations. 

The VECTOR_DISTANCE function is a new feature of the SQL Database in fabric that can calculate the distance between two vectors enabling similarity searching right in the database. You will be using this function in samples as well as in the RAG chat application; both utilizing the vectors you just created for the Products table.


declare @search_text nvarchar(max) = 'I am looking for a red bike and I dont want to spend a lot'
declare @search_vector vector(1536)
exec dbo.create_embeddings @search_text, @search_vector output;
SELECT TOP(4) 
p.ProductID, p.Name , p.chunk,
vector_distance('cosine', @search_vector, p.embeddings) AS distance
FROM [SalesLT].[Product] p
ORDER BY distance

<img width="1275" height="609" alt="image" src="https://github.com/user-attachments/assets/dfa44eee-56cb-4ed9-be8d-02ed6ce20251" />

create or alter procedure [dbo].[find_products]
@text nvarchar(max),
@top int = 10,
@min_similarity decimal(19,16) = 0.80
as
if (@text is null) return;
declare @retval int, @qv vector(1536);
exec @retval = dbo.create_embeddings @text, @qv output;
if (@retval != 0) return;
with vector_results as (
SELECT 
        p.Name as product_name,
        ISNULL(p.Color,'No Color') as product_color,
        c.Name as category_name,
        m.Name as model_name,
        d.Description as product_description,
        p.ListPrice as list_price,
        p.weight as product_weight,
        vector_distance('cosine', @qv, p.embeddings) AS distance
FROM
    [SalesLT].[Product] p,
    [SalesLT].[ProductCategory] c,
    [SalesLT].[ProductModel] m,
    [SalesLT].[vProductAndDescription] d
where p.ProductID = d.ProductID
and p.ProductCategoryID = c.ProductCategoryID
and p.ProductModelID = m.ProductModelID
and p.ProductID = d.ProductID
and d.Culture = 'en')
select TOP(@top) product_name, product_color, category_name, model_name, product_description, list_price, product_weight, distance
from vector_results
where (1-distance) > @min_similarity
order by    
    distance asc;
GO


Next, you need to encapsulate the STORED PROCEDURE into a wrapper so that the result set can be utilized by our GraphQL endpoint. Using the WITH RESULT SET syntax allows you to change the names and data types of the returning result set. This is needed in this example because the usage of sp_invoke_external_rest_endpoint and the return output from extended stored procedures

create or alter procedure [find_products_api]
@text nvarchar(max)
as 
exec find_products @text
with RESULT SETS
(    
    (    
        product_name NVARCHAR(200),    
        product_color NVARCHAR(50),    
        category_name NVARCHAR(50),    
        model_name NVARCHAR(50),    
        product_description NVARCHAR(max),    
        list_price INT,    
        product_weight INT,    
        distance float    
    )
)
GO
You can test this newly created procedure to see how it will interact with the GraphQL API by running the following SQL in a blank query editor in Microsoft Fabric:

exec find_products_api 'I am looking for a red bike'

<img width="1237" height="452" alt="image" src="https://github.com/user-attachments/assets/a026c66f-bfcf-4d18-a252-7a45c98ddd47" />




