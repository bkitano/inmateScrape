# inmateScrape

## Background
While the Federal Bureau of Prisons (BOP) provides an inmate [*locator*](https://www.bop.gov/mobile/find_inmate/#inmate_results), where you can search for individual inmates, it doesn't provide an inmate database, where you can query for inmates in aggregate. This makes analysis of the federal prison population intractable. I decided to build such a database.

To search for an inmate in the BOP, the locator requires an Inmate Registry Number (IRN), which is a five digit inmate number and a three digit region number. There are therefore 100M possible IRNs. In order to scrape every actual inmate, I would need to brute force guess every possible register number. This would take too long to do on my local machine, so I decided to leverage the AWS Free Tier and parallelize my guesses via SQS and Lambda, and write to DynamoDB.

### Other datasets
- [Crime and Incarceration in the United States - Kaggle](https://www.kaggle.com/christophercorrea/prisoners-and-crime-in-united-states?select=ucr_by_state.csv)
- (Incarceration Trends Dataset)[https://github.com/vera-institute/incarceration-trends]
- [Census of State and Federal Adult Correctional Facilities, 2005 (ICPSR 24642)](https://www.icpsr.umich.edu/web/NACJD/studies/24642/versions/V3/datadocumentation#)
- [Bureau of Justice Statistics](https://www.bjs.gov/rawdata.cfm)

## Ethics
In creating this dataset, we must consider challenging ethical questions:

How can we respect the rights of inmate, a vulnerable population?
What does releasing this dataset enable that was not previously possible, and does it pose an unreasonable risk of harm?
What is the role of consent in collecting data from inmates who by design don't have a voice?

## Architecture
I created an AWS solution to scrape the Federal Bureau of Prisons Inmate Locator.

### 1. Expose the BOP search API.
The BOP API for searching inmate registry numbers is a simple GET request:

```
response = requests.get("https://www.bop.gov/PublicInfo/execute/inmateloc?todo=query&output=json&inmateNum=11111-001&inmateNumType=IRN")
```

### 2. Create a Lambda function to read from SQS and hit the BOP API.
I initially wanted to run the scrape locally using the `multiprocessing` module to run all of the guesses in parallel, but it would still take too long to resolve every request in its own process. Therefore, it made more sense to me to leverage [AWS Lambda](https://aws.amazon.com/lambda/) to run batches of guesses in parallel.

The requirements of the Lambda function are simple: I need to provide to it the range of IRNs, and it needs to write valid IRNs and their attributes to DynamoDB. To test my Lambda function, I used an API Gateway trigger which I could hit using GET requests. I used the minimum required memory for each Lambda instance (128MB) and set the timeout to be 30s.

One caveat is that you have to upload a .zip file to Lambda in order to import modules. You can do this via `pip install -t . <your_module>` and then `zip -r . ../myLambdaModule.zip`.

### 3. Batch guesses to store as messages in SQS.
To scale up the operation to the 100M possible IRNs, I initially wanted to use a for loop to send off a bunch of Lambda triggers. However, I realized that even if I made the batch size 1k guesses, it would still take 100k iterations to complete all the requests, which would mean I would have to leave my computer open to finish. In addition, as you increase the batch size, the longer the Lambda function takes to execute; this may lead to failures if the Lambda timeout setting is too low.

Instead, I decided to use [Amazon SQS](https://aws.amazon.com/sqs/), where I could send off as many batches as I wanted without having to wait for them to complete. I could also set the batch size to be small enough to avoid timeout failures. In addition, an added benefit is the built in retry logic of SQS and Lambda: if the Lambda function fails, the SQS message gets added back to the queue to try again. The only minor complication is the relation between the Lambda timeout limit and the SQS visibility timeout. According to the [docs](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html):

> To allow your function time to process each batch of records, set the source queue's visibility timeout to at least 6 times the timeout that you configure on your function. The extra time allows for Lambda to retry if your function execution is throttled while your function is processing a previous batch.

I generate the batches and write them all to SQS in the `kickoff.ipynb` notebook.

### 4. Store the valid guesses in DynamoDB.
[DynamoDB](https://aws.amazon.com/dynamodb/) seemed like the right way to store this data. Despite it being unstructured, a relational database with a requisite server connection seemed unnecessarily complicated. In retrospect, I would have increased the concurrent write limits to enable more Lambda functions to write to the database at the same time.

Costs
In total, I used ~427k Lambda GB-Seconds, which amounted to $0.46 after the Free Tier credits. The DynamoDB and SQS are totally within the Free Tier.

Data
In total, there were ~280k inmates listed as of 6/23/2020, about 61MB.

Each inmate entry has the following attributes:

- Name
- Age
- Race
- Sex
- Release Date
- Inmate ID
- Location

I've removed the personally identifiable information (PII) from this data by taking only a subset of the columns, removing their names and inmate ids, so we can study the joint distributions of inmates while respecting currently or previously incarcerated people's privacy. However, it is likely that an adversary could determine the identity of the entries.

## Next Steps
### Updating the database at regular intervals

### Assessing Segregation in the federal prison system
So far, I've determined that Native Americans are dramatically overrepresented in the federal prison system.


| race | capita |
| ------ | ----- |
| American Indian | 0.000228 | 
| Asian | 0.000012 |
| Black | 0.000143 |
| White | 0.000057 |