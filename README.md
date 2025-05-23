# Building a district problem solution capability
For an overview of this capability, please see the informational overview powerpoint
@ School Board Meeting Transcript Problem Solution.pdf 

## **File Documentation**

**step0\_data\_prep\_and\_relevancy\_filtering.ipynb**

* **Files required to run this file:**  *districtview2025.topics.jsonl* \- district view jsonl file containing at least the following columns:  
  * *centroid\_lat, centroid\_lon, leaid, place\_name, state\_name, video\_id, caption\_sentences, caption\_sentence\_topics*  
* **Outputs:**  *deepseek\_kw\_chunks\_2025.csv*  
  * File containing the following columns: *centroid\_lat, centroid\_lon, leaid, place\_name, state\_name, video\_id, caption\_sentences, caption\_sentence\_topics, initial\_keywords, kw\_text\_chunks, relevancy\_category, indices, kept\_sections*  
  * See step 7 in Method below to view the data dictionary for this file  
* **Purpose:** Clean data, break transcripts into sections and identify relevant sections to a school board discussion topic  
* **Method:**  
1. Load distictview jsonl data  
2. Clean and remove invalid data:   
   * Remove music tags, empty transcripts, transcripts with less than 5 minutes worth of words (\<600 words).   
   * Perform text cleaning ,including punctuation and case standardization   
3. Use topic keywords from the literature review to filter meetings to those containing at least 1 keyword  
   * For school closures, this includes the following words: *closure, closing, merging, merge, consolidate, consolidation, reassignment, utilization, building repurposing, right sizing*  
4. Generate context windows around keyword locations  
   * windows are at least 30 sentences long (15 sentences on each side of an identified keyword). *Note: Overlapping or consecutive windows are merged into a single “transcript section” to avoid overrepresentation*  
   * Note: Windows are referenced as keyword chunks throughout the files  
5. Defines a relevancy tagging prompt which returns one of the following categories  
   * Categories a context window can be tagged as: CovidRelatedSchoolClosure, LongTermSchoolClosure, NonCovidTemporarySchoolClosure, NoneOrUnrelated  
   * See prompt in the *check\_relevancy()*  function body, denoted as “prompt”  
6.  Send defined LLM prompt through Deepseek (through ollama) to tag context window sections  
   * Response from DeepSeek is parsed according to prompt instructions for expected output. If a response is incorrectly returned from DeepSeek, a f string is reaturned as follows: 

     f"INCORRECT RESPONSE RETURNED: {full\_deepseek\_response}"

   * A timeout procedure is implemented to ensure a single section exits upon exceeding 5 minutes of computation time. Implemented due to extended and unending periods of DeepSeek thinking time   
7. Save new csv where rows represent individual meetings, and columns are the following:  
   * 'centroid\_lat': district latitude  
   * 'centroid\_lon':  district longitude  
   * 'leaid': district unique id  
   * 'place\_name': district name  
   * 'state\_name': district state acrynomym  
   *  'video\_id': unique video id  
   * 'caption\_sentences': transcript represented as a list of sentences  
   *  'caption\_sentence\_topics': district view clustering sentence topics represented as integers  
   * 'initial\_keywords': list of manually defined keywords found to be in the meeting (from step 3 in this Method list)  
   * 'kw\_text\_chunks': list of strings where each string represents a single context window where an initial keyword was found  
   * 'relevancy\_category': list of DeepSeek tagged relevancy categories whose element indices correspond with the meeting section index (ex: if index 0 is denoted as CovidRelatedSchoolClosure, this means the first section of the meeting has been tagged as related to Covid closure and is therefore not relevant to our study)  
   * 'indices': list of integers representing the indices of DeepSeek tagged *relevant* sections   
   *  'kept\_sections': list of strings representing context windows corresponding exactly to the relevant section indices \- ie these are sections of meetinbgs which are relevant to the topic  
   * 'meeting\_date': date of meeting

**step1\_problem\_solution\_gpt\_annotation.ipynb**

* **Files required to run this file:** deepseek\_kw\_chunks\_2025.csv   
  * File containing the following columns: *centroid\_lat, centroid\_lon, leaid, place\_name, state\_name, video\_id, caption\_sentences, caption\_sentence\_topics, initial\_keywords, kw\_text\_chunks, relevancy\_category, indices, kept\_sections*  
  * See step0\_data\_prep\_and\_relevancy\_filtering.ipynb step 7 for the data dictionary of this file  
* **Outputs:** *PROBLEM\_SOLUTION\_GPT4o\_JSON\_RESULTS/*

  *GPT4o\_output\_deepseek\_results\_24\_25\_only.csv.json*

  * JSON file whose elements each represent the problem solution annotation of a single section of a meeting. Fields for each element include video\_id and  problem\_solution\_dictionary:  
    * video\_id: unique id of the video the section is pulled from  
    * problem\_solution\_dictionary: GPT4.1 generated dictionary according to engineered prompt where keys are problems discussed in the section and associated values are problems addressing respective problems  
* **Purpose:** Load and flatten relevant sections from relevancy tagging, filter to desired dates, annotate sections to get problem solution information as JSON  
* **Method:**  
1. Load DeepSeek output from step0\_data\_prep\_and\_relevancy\_filtering.ipynb file  
2. Filter data to desired dates  
   * Remove NaT and invalid dates (ex: dates which begin with 0, dates which cannot be parsed to contain year, month and day such as a single random int like 123\)  
3. Define problem solution prompt  
   * Defines problem solution prompt with instructions, example output, & required JSON output  
   *  Appends section text to instruction for final prompt construction and ensures final prompt with text is within length of max tokens by truncating  text centrally and only as needed to avoid cropping prompt  
4. Pass prompt to GPT4.1 model for annotation for every relevant section  
   * Validate output from LLM according to prompt, checking JSON loadability and required key existence  
5. Save JSON problem solution annotation results to single JSON in *PROBLEM\_SOLUTION\_GPT4o\_JSON\_RESULTS* directory with filename *GPT4o\_output\_deepseek\_results\_24\_25\_only.csv.json*

**step2\_problem\_solution\_data\_prep\_and\_aggregation.ipynb**

* **Files required to run this file:**   
  * *GPT4o\_output\_deepseek\_results\_24\_25\_only.csv.json* \- output of *step1\_problem\_solution\_gpt\_annotation.ipynb* file   
  * *deepseek\_results\_24\_25\_only.csv* \- DeepSeek results from step0\_data\_prep\_and\_relevancy\_filtering.ipynb which are filtered to relevant dates (for school closures, pulled most recent data from 2024-25), used for metdata  
* **Outputs:**   
  * *district\_problems.json* \- LLM discovered problem data dictionary  
  * *prob\_solution\_50\_district\_example.csv* \- problem solution tagged results of first 50 districts where rows represent a single unique district-problem pair containing the following columns:   
    * *leaid*: the district id   
    * *problem\_tag*: which problem tag the solutions are associated with for this district-problem group  
    * *solutions\_list*: list of solutions this district has discussed specific to the problem specified  
* **Purpose:** Prepare and clean problem solution annotated section data, aggregate problem solution data by meeting and district, build discrete problem data dictionary, tag problem strings with data dictionary problem topics  
* **Method:**  
1. Load DeepSeek data for original district view meta data access, load problem solution output from *step1\_problem\_solution\_gpt\_annotation.ipynb* file   
2. Clean and prep data data for aggregation  
   * Remove sections which don’t contain problem solution information  
3. Generate meeting aggregated dataframe   
   * concatenate problem solution dictionaries for those with same meeting id  
4. Generate district aggregated dataframe from meeting dataframe  
   * concatenate problem solution dictionaries for those with same district id  
5. Define problem solution data dictionary prompt  
   * Have it extract top n most represented topics from the given list  
   * See tag in *get\_problem\_solution\_topics()* function body  
6. Get the flattened list of all problems (problem solution dictionary keys) across all districts  
7. Feed prompt with appended problem list from step 7 to GPT4.1 model   
8. Save problem data dictionary  as *district\_problems.json* efines prompt to tag a given string with one of the problem topics  
9.  Define a prompt to tag input strings using the discovered problem data dictionary  
   * when tag "other" is assigned to strings, this denotes the problem as outside the scope of N relevant problems  
   * See tag in *get\_problem\_tag()* function body  
10.  Feed problem tagging prompt to GPT4.1 to categorize each problem string in each district problem solution dictionary  
11. Save the results in *prob\_solution\_50\_district\_example.csv* (used only first 50 districts as proof of concept) whose columns are   
    * *leaid*: the district id   
    * *problem\_tag*: which problem tag the solutions are associated with for this district-problem group  
    * *solutions\_list*: list of solutions this district has discussed specific to the problem specified

**step3\_ps\_matching.ipynb**

* **Files required to run this file:**   
  * *deepseek\_results\_24\_25\_only.csv* \- DeepSeek results from step0\_data\_prep\_and\_relevancy\_filtering.ipynb which are filtered to relevant dates (for school closures, pulled most recent data from 2024-25), used for metdata  
  * *prob\_solution\_50\_district\_example.csv* \- output from *step2\_problem\_solution\_data\_prep\_and\_aggregation.ipynb*   
* **Outputs:** *sample\_district\_matching.csv \-* example of district matching output, two examples for each problem tag  
* **Purpose:** Cleans problem tags, constructs problem matrices generates semantics oriented embeddings for solutions, provides problem solution matching, and identifies district problems which have no current solutions  
* **Method:**   
1. Load case example \- *prob\_solution\_50\_district\_example.csv* and metadata file  
2. Clean problem tags of syntactic issues  
   * Fix double quotes and leading/trailing quotes from parsking LLm output  
3. Create a new dataframe by grouping by district and problem tag, consolidating solutions of those in the same group into a list og solutions associated with the given district and problem tag  
4. Merge with metadata file to get district place information  
5. Generate problem dataframe by grouping by problem tag  
6. Find solution vectors to represent a single district’s solution approaches to given problem tag  
   * Use sentence transformers to get vectors for each solution element in a solution list, average to get vector representation of original text solutions list  
7. Identify districts-problems with no solutions into variable MISSING\_SOLUTIONS  
8. Implement district matching  
   * use cosine similarity on district solution vectors to get most similar and most dissimilar district for a given problem tag. (Note: why get most dissimilar district solution? It is the differences in solutions that can be the most useful in building new solutions or providing new viewpoints\!)  
9. Save result of step 8 into csv containing the following columns:  
   * *which\_problem*: the problem tag being discussed  
   * *this\_district\_leaid*: which district being matched  
   * *this\_district\_place*: which district to match’s name  
   * *this\_district\_state*: which district to match’s state acronym   
   * *this\_district\_solution*: text solution approach of the district being matched to the specified problem  
   * closest\_leaid: district id of the district with the most similar solution to the one being matched  
   * *closest\_place*:  name of the district with the most similar solution to the one being matched  
   * *closest\_state*: state acronym of the district with the most similar solution to the one being matched  
   * *closest\_solution*: text solution provided by the district with the most similar solution to the one being matched  
   * farthest\_leaid: district id of the district with the least similar solution to the one being matched  
   * *farthest\_place:* name of the district with the least similar solution to the one being matched  
   * farthest\_state:  state acronym of the district with the least similar solution to the one being matched  
   * *farthest\_solution*: text solution provided by the district with the least similar solution to the one being matched

## **IRB Documentation**

**Participant Information Sheet**  
PDF write up for IRB participant information sheet with research subject “Understanding School Board Engagement with Education Policy Strategies”

**Example Survey**  
Example survey created regarding school closure topics for board member insight  
[https://docs.google.com/forms/d/e/1FAIpQLSfUwi8lGmwrVA15rQsDRz4YGl8UPqGbE8q2lCM7CJqcS\_O\_VA/viewform?usp=sharing\&ouid=115032336610307901975](https://docs.google.com/forms/d/e/1FAIpQLSfUwi8lGmwrVA15rQsDRz4YGl8UPqGbE8q2lCM7CJqcS_O_VA/viewform?usp=sharing&ouid=115032336610307901975) 

## **Quality Check Documentation**

**expected\_relevancy\_results\_100.xlsx**  
Excel file with quality check information for DeepSeek relevancy tagging. The description is as follows: Sections from a random selection of meetings are manually annotated until there are 50 expected "relevant", and 50 expected "non relevant" sections. Because each video is independent and there is no inherent ordering to the rows in each file, the first 10-15 unique video ids (aka rows) are arbitrarily selected from each deepseek result test file until the 50-50 ratio is achieved by manual annotation. The sections are then passed through ChatGPT model (gpt4o) for auto tagging. The results are compared and accuracy of deepseek tagging and gpt relevancy tag is calculated and confusion values are noted.			  
			  
**gpt4o\_text\_annotation\_check.xlsx**  
Excel file with quality check information for ChatGPT annotation of detailed topic analysis of school closure topic regarding closure action plan steps, reasoning for closure, urgent boolean, implementation concerns, closure specific policies mentioned, sentiment towards closure	expressed, expected outcome effects of closure, decision considerations in closure discussion, infrastructure outcome post closure, and closure status of the closure being discussed.

A random number generator was used with range \[0,len(file)\] from https://www.random.org/ to generate a random int to index random row of each month\_year file to annotate as accuracy check. Because of complexity regarding specificity of results for each field, a tag is deemed correct soley on the following information:  
(1) if it does not contain hallucinations and   
(2) it answers the field with accurate information explicitly reflecting information from the text section. This means the quality evaluation does not reflect a given response's potential for specificity and is soley concerned with the accuracy of the actual response contents.   
Note: an empty tag is incorrect if there is explicit information included in the text relevant to the tag (see (2)). Notes are included in bolded red text.

			
			
