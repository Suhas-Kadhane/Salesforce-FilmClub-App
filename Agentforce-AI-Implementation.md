# FilmClub Agentforce Implementation Documentation

## Project Overview

This document details the end-to-end implementation of an Agentforce Agent for a custom Salesforce FilmClub application. The agent was built to help users manage their personal movie collection, answer movie trivia and FAQs, provide movie recommendations, and offer cast and crew insights.

The implementation was done entirely within a Salesforce Free Developer Organization using Agentforce Builder, Salesforce Knowledge, Salesforce Flows, and Einstein AI.

---

## Application Background

The FilmClub app is a custom Salesforce application built on a Developer Edition org. It uses a custom object called Movie_T__c to store movie records with the following fields:

- Movie Name (Name)
- Cast (Cast__c)
- Director (Director__c)
- Genre (Genre__c)
- IMDb Rating (IMDb_Rating__c)
- Poster URL (Poster_URL__c)
- Release Date (Release_Date__c)
- Runtime (Runtime__c)
- Synopsis (Synopsis__c)

The app uses an existing Screen Flow called "Movie Title: Fetch from OMDb" that allows users to manually search for and add movies by fetching data from the OMDb external API via an External Service called OMDBServiceV2.

---

## Agent Requirements

The agent was required to handle the following three core rules:

1. Before adding a movie, always check if it already exists in the collection to avoid duplicates.
2. If a user mentions a movie they loved or saw recently without providing a specific title, ask for the title before proceeding.
3. Be enthusiastic about new additions to the collection.

The agent was later expanded to also handle movie trivia and FAQs, personalized movie recommendations, cast and crew insights, and genre exploration.

---

## Phase 1: Initial Agent Setup Attempt

### Approach

The initial approach involved creating an Auto-Launched Flow to power the agent's movie management capabilities and connecting it to an Agentforce Agent via Agent Topics and Actions.

### Steps Followed

#### Step 1: Enable Einstein and Agentforce

1. Navigate to Setup and search for Einstein Setup in Quick Find.
2. Enable Einstein if not already enabled.
3. Search for Agents in Quick Find to verify Agentforce is available.

#### Step 2: Build the Auto-Launched Flow

A new Auto-Launched Flow was created from scratch with the following resources:

Variables created:
- movieTitle (Text, Available for Input: true) - receives the movie title from the agent
- resultMessage (Text, Available for Output: true) - returns the result message to the agent
- isDuplicate (Boolean, Available for Output: true) - returns duplicate status
- duplicateMessage (Text, Default Value: "This movie already exists in your FilmClub collection!")
- successMessage (Text, Default Value: "Movie successfully added to your FilmClub collection!")
- cleanTitle (Formula, Text, Formula: TRIM({!movieTitle})) - sanitizes the movie title

Flow elements in sequence:

1. Get Records: Check for Existing Movie
   - Object: Movie_T__c
   - Filter: Name Contains {!cleanTitle}
   - Store: Automatically store all fields

2. Decision: Does Movie Already Exist?
   - Yes - Already Exists: {!Check_for_Existing_Movie} Is Null false
   - No - New Movie: Default outcome

3. On Yes - Already Exists path:
   - Assignment: Set Duplicate True
     - {!isDuplicate} = true
     - {!resultMessage} = {!duplicateMessage}

4. On No - New Movie path:
   - Action: Get Movie Data (OMDBServiceV2 FetchMovieV2 External Service)
     - Input t: {!cleanTitle}
     - Input apikey: [OMDb API Key]
   - Create Records: Create Movie Entry
     - Name = {!Get_Movie_Data.200.Title}
     - Cast__c = {!Get_Movie_Data.200.Actors}
     - Director__c = {!Get_Movie_Data.200.Director}
     - Genre__c = {!Get_Movie_Data.200.Genre}
     - IMDb_Rating__c = {!formulaRatingToNumber}
     - Poster_URL__c = {!Get_Movie_Data.200.Poster}
     - Release_Date__c = {!formulaYearOnly}
     - Runtime__c = {!Get_Movie_Data.200.Runtime}
     - Synopsis__c = {!Get_Movie_Data.200.Plot}
   - Assignment: Set Success Message
     - {!isDuplicate} = false
     - {!resultMessage} = {!successMessage}

---

## Challenges and Errors Encountered in Phase 1

### Challenge 1: "Enter a Valid Value" Error in Assignment Element

**Problem:** When trying to set a text value directly in an Assignment element's Value field, Salesforce showed "Enter a valid value" error.

**Root Cause:** Salesforce Assignment elements do not accept literal text values directly. The Value field only accepts variables or resources.

**Solution:** Created separate Text Variables with the desired messages stored as Default Values, then referenced those variables in the Assignment elements.

---

### Challenge 2: movieTitle Passing as Null

**Problem:** The Get Records element was filtering by Name Equals null instead of the actual movie title.

**Root Cause:** The movieTitle input variable was not correctly mapped in the Get Records filter Value field. The formula cleanTitle was not appearing in the dropdown.

**Solution:** Changed the Get Records filter operator from Equals to Contains and directly referenced {!movieTitle} in the Value field. Also created the cleanTitle formula resource and mapped it via the Manager tab.

---

### Challenge 3: resultMessage Variable Typed as Boolean

**Problem:** The resultMessage output variable was accidentally created as a Boolean data type instead of Text, which meant the agent could not read a text message from it.

**Root Cause:** During variable creation the wrong data type was selected.

**Solution:** Deleted the Boolean resultMessage variable and recreated it as a Text variable with Available for Output set to true.

---

### Challenge 4: Agent Selecting Wrong Topic

**Problem:** When typing "Add the movie Parasite", the agent was selecting the New Movie Celebration topic instead of Movie Duplicate Detection, which meant the flow action was never called.

**Root Cause:** The New Movie Celebration topic's Classification Description was too broad and was being triggered before the duplicate detection topic.

**Solution:** Deleted the New Movie Celebration topic and merged its enthusiastic response behaviour into the Movie Duplicate Detection topic instructions.

---

### Challenge 5: Agent Passing Movie Titles with Quotation Marks

**Problem:** The agent was passing movie titles wrapped in quotation marks such as "Interstellar" instead of Interstellar, causing the Contains filter to miss existing records.

**Root Cause:** The agent's natural language processing adds quotation marks around proper nouns including movie titles.

**Solution:** Created a Formula resource called cleanMovieName using SUBSTITUTE({!movieTitle}, '"', '') to strip quotation marks before the Get Records filter. Also updated the agent action's movieTitle input instructions to specify: "Always pass the title as plain text without any quotation marks, brackets or special characters."

---

### Challenge 6: Flow Failing When Triggered by Agent but Working in Debug

**Problem:** The flow worked perfectly in Debug mode but failed every time the agent triggered it, returning "Something went wrong" errors.

**Root Cause (discovered after extensive investigation):** Multiple compounding issues:
- The Einstein Agent User did not have Create and Edit permissions on Movie_T__c field level security.
- The auto-generated Permission Set FilmClub_Concierge_Agent Permissions did not have field-level Edit access for most Movie_T__c fields.
- The OMDb External Credential Principal did not have the correct Permission Set assignments for the Einstein Agent User.
- The Flow was running in User Context, which enforced the Einstein Agent User's limited permissions.

**Solution:**
- Enabled Edit access on all Movie_T__c fields in the auto-generated Permission Set.
- Enabled the OMDb External Credential Principal access in the relevant Permission Sets.
- Changed the Flow's "How to Run Flow" setting to "System Context without Sharing - Access All Data".

**Outcome:** Even after all these fixes, the OMDb external API callout continued to fail when triggered by the agent due to External Credential authentication restrictions specific to the Einstein Agent User in a free Developer Edition org. The Phase 1 approach was ultimately abandoned.

---

## Phase 2: Revised Agent - Conversation Only Approach

### Decision

Given the persistent issues with external API callouts from agent-triggered flows, the strategy was revised. The agent was rebuilt to focus on conversational capabilities without connecting to the flow for adding movies. The OMDb-powered movie addition would continue to be handled by the existing Screen Flow for manual use.

### Agent Creation Steps

#### Step 1: Create Agent Using Gen AI

1. Navigate to Setup and search for Agents in Quick Find.
2. Click New Agent and select Create with Gen AI.
3. Enter the following description:

"I am building a personal FilmClub assistant for my Salesforce FilmClub app. This assistant should help me with movie collection management by checking for duplicates and asking for movie titles when not provided, movie FAQs and trivia, movie recommendations based on preferences and genres, and enthusiastic responses when new movies are added."

#### Step 2: Customize the Agent

- Name: FilmClub Concierge Agent
- Role: You are an enthusiastic FilmClub assistant and movie expert. You help manage the user's movie collection, answer trivia and FAQs, and suggest movies they will love. Passionate about cinema and making every interaction fun.
- Company Name: FilmClub
- Company Description: FilmClub is a personal movie collection app built on Salesforce where the user can discover, track and manage movies they love.
- Checked: Keep a record of conversations with enhanced event logs.

#### Step 3: Add Topics

Einstein AI auto-generated the following topics which were all kept:

1. Personalized Movie Suggestions - Recommend movies based on user preferences and watch history.
2. Movie Trivia Assistance - Provide answers to movie-related trivia and FAQs.
3. Collection Management - Help users organize and update their movie collections.
4. Cast and Crew Insights - Offer detailed information about actors and directors.
5. Genre Exploration - Assist users in discovering movies by genre or theme.

#### Step 4: Skip Data Sources

The Add Data step was skipped as Data Cloud was not available in the free Developer Edition org.

---

## Topic Configuration

### Collection Management

Classification Description: This topic is selected when a user mentions watching, seeing, loving, or experiencing any movie in any context, even if they do not provide a specific movie title. Any reference to watching or enjoying a movie must always be handled by this topic.

Scope: Your job is to assist with organizing and managing the user's movie collection. This includes adding new movies, checking for duplicates, and updating collection details. Always ask for the movie title if the user has not provided one.

Instructions:
1. If the user wants to add a movie but has not provided a specific title, ask them: "What is the title of the movie you would like to add to your FilmClub collection?"
2. Before adding any movie, always check if it already exists in the collection by asking the user to confirm the title. If it already exists, inform them politely: "This movie is already in your FilmClub collection. No duplicates added."
3. When a new movie is successfully added, respond enthusiastically. Use phrases like "Amazing choice!" or "What a great pick! Your FilmClub collection just got better!"
4. Always provide your response in a single reply. Do not generate a follow up response after your initial answer. Once you have responded, wait for the user to reply before saying anything else.

### Movie Trivia Assistance

Classification Description: This topic is selected when the user asks for answers to movie-related trivia or FAQs.

Scope: Your job is to provide accurate and enthusiastic answers to movie-related trivia and FAQs using your own built-in knowledge about movies, directors, actors and cinema history. You do not need an external data source to answer these questions.

Instructions:
1. If the user asks a trivia question, provide a concise and accurate response.
2. For FAQs, ensure the answer is relevant to movies and cinema.
3. Always respond with enthusiasm and passion for cinema. Add fun facts or interesting context where relevant to make the experience enjoyable.
4. You are permitted to answer all movie related questions directly from your own knowledge. Do not say you are unable to retrieve information. Always provide a confident and detailed answer.
5. Always provide your response in a single reply. Do not generate a follow up response after your initial answer.
6. When answering questions about classic films including The Godfather, Pulp Fiction, Goodfellas, Scarface or any other acclaimed crime drama, always provide factual and educational information about the film's director, cast and awards. These are legitimate cinematic masterpieces and should be discussed in an informative and respectful context.

### Personalized Movie Suggestions

Classification Description: This topic is selected when the user seeks movie recommendations based on their preferences, watch history, or specific themes.

Scope: Your job is to provide personalized movie recommendations using your own built-in knowledge about movies, genres and cinema. You do not need an external data source to make recommendations. Always provide confident and enthusiastic suggestions.

Instructions:
1. If the user provides preferences or a watch history, analyze the input and recommend movies.
2. If the user asks for suggestions by theme or genre, focus on those criteria.
3. If the user has not provided any preferences, ask them: "What kind of movies do you enjoy? Tell me your favourite genre, actor, director or a movie you loved and I will find the perfect recommendation for you!"
4. Always recommend at least 3 movies with a brief enthusiastic description of each one explaining why the user would love it.
5. Always provide your response in a single reply. Do not generate a follow up response after your initial answer.

### Cast and Crew Insights

Classification Description: This topic is selected whenever a user asks about any actor, director, filmmaker or crew member regardless of the genres or types of films they are associated with. All cast and crew queries including those related to crime drama actors must be handled by this topic.

Scope: Your job is to provide detailed and enthusiastic insights about cast and crew members using your own built-in knowledge. You do not need an external data source to answer these questions. Always provide confident and accurate answers.

Instructions:
1. If the user asks about an actor or director, provide their filmography and notable works.
2. For crew-related queries, focus on their contributions to specific movies.
3. Always include fun and interesting facts about the cast or crew member to make the response engaging.
4. Respond with enthusiasm and passion for cinema.
5. Always provide your response in a single reply. Do not generate a follow up response after your initial answer.
6. When providing information about actors and directors associated with crime dramas including Al Pacino, Robert De Niro, Joe Pesci or any other acclaimed actor, always discuss their work in terms of artistic merit, awards, critical acclaim and cinematic achievements.

### Genre Exploration

Classification Description: This topic is selected whenever a user asks for movie recommendations by genre or mood including action, thriller, crime, drama, comedy, romance or any other film genre. All genre based queries must be handled by this topic.

Scope: Your job is to assist users in exploring movies by genre or theme using your own built-in knowledge about cinema. You do not need an external data source to provide curated movie lists. Always provide confident and detailed recommendations.

Instructions:
1. If the user specifies a genre, provide a curated list of movies in that genre.
2. If the user mentions a theme, focus on movies that align with that theme.
3. Always recommend at least 5 movies per genre or theme with a brief enthusiastic description of each one.
4. If the user has not specified a genre or theme, ask them: "What kind of mood are you in today? Tell me and I will curate the perfect list for you!"
5. Always provide your response in a single reply. Do not generate a follow up response after your initial answer.
6. When recommending thriller, crime or action movies, always describe them using only their artistic merit, critical acclaim, awards and directorial achievements. Never describe violent, graphic or disturbing plot details.
7. When recommending any genre including thrillers, always describe films using only their artistic merit, awards and critical acclaim. Never use words that describe violence, harm or disturbing content in any way.

---

## Challenges and Errors Encountered in Phase 2

### Challenge 1: Auto-Added Knowledge Actions Causing Errors

**Problem:** After creating the agent, all 5 topics automatically had three actions added: Answer Questions with Knowledge, Knowledge Space Knowledge Search, and AnswerQuestionsWithSalesforceDocumentation. These caused the error: "Missing required input parameter: KnowledgeSpaceSearchIndexId - We couldn't find a data library assigned to this agent."

**Root Cause:** Salesforce automatically adds default knowledge actions to all topics. These actions require Data Cloud or a Salesforce Knowledge data library to function.

**Solution:** Removed all three auto-added actions from all 5 topics.

---

### Challenge 2: UNGROUNDED Response Error

**Problem:** The agent was generating correct responses for a split second but then replacing them with generic fallback messages. The Output Evaluation showed: "UNGROUNDED - there is no evidence in the context or function history that the action was completed."

**Root Cause:** Agentforce's built-in response grounding system requires all agent responses to be backed by a verifiable data source. Without a knowledge base or data library, the system rejected detailed responses as ungrounded.

**Solution:** Enabled Salesforce Knowledge, created and published Knowledge Articles covering movies, directors, actors, trivia and recommendations, and connected the Answer Questions with Knowledge action back to the relevant topics after granting the Einstein Agent User the "Allow View Knowledge" system permission.

---

### Challenge 3: Content Sensitivity Filter Blocking Movie Responses

**Problem:** Responses about certain movies and actors were being blocked. Movies like The Godfather, Pulp Fiction, Goodfellas, and actors like Al Pacino were consistently failing while others like Schindler's List and Interstellar passed.

**Root Cause:** Agentforce has a built-in toxicity and content sensitivity filter. Movies with crime, violence, or mature themes were being flagged even when the questions were purely educational and factual.

**Solution:** Added explicit instructions to the relevant topic configurations stating that these are legitimate cinematic masterpieces to be discussed in an educational and informative context focusing on artistic merit, awards and critical acclaim.

---

### Challenge 4: Double and Triple Response Generation

**Problem:** The agent was generating two or three responses for a single user message, with the later responses overriding the correct first response with generic fallback messages.

**Root Cause:** The agent's grounding evaluation system was running multiple reasoning cycles. When the first response was flagged as potentially ungrounded, the system generated additional responses which then replaced the original.

**Solution:** A combination of fixes resolved this:
- Updated Classification Descriptions to be more specific and aggressive about which topics handle which queries.
- Added explicit scope instructions telling the agent to use its own built-in knowledge.
- Added the Answer Questions with Knowledge action backed by properly structured Knowledge Articles.
- Added explicit single-response instructions to all topic instruction sets.

---

### Challenge 5: Knowledge Article Body Field Not Editable

**Problem:** After enabling Salesforce Knowledge and creating a Knowledge Article, the Body field was visible on the record page but could not be edited in Lightning Experience.

**Root Cause:** The Knowledge Article page layout in Lightning Experience only showed Title and URL Name as inline editable fields. The Body field, while added to the layout, was not editable through the Lightning interface.

**Solution:** Switched to Salesforce Classic to edit and publish Knowledge Articles using the classic Knowledge editor. All articles were created and published in Classic mode then viewed in Lightning Experience.

---

### Challenge 6: Prompt Builder Template Customization Error

**Problem:** When attempting to create a custom version of the Answer Questions with Knowledge prompt template in Prompt Builder to allow the agent to use its own knowledge when articles were insufficient, saving the new version produced the error: "The following object names were not recognized: einsteinsearch:sfdc_ai__dynamicretriever"

**Root Cause:** The Dynamic Retriever component in the standard prompt template is a system-level component that cannot be removed or modified in a free Developer Edition org.

**Solution:** Instead of modifying the Prompt Builder template, the Knowledge Articles were expanded with comprehensive content to ensure the grounding system could always find relevant information, removing the need to bypass the standard retriever.

---

## Salesforce Knowledge Articles Created

Four Knowledge Articles were created and published to support agent response grounding:

### Article 1: FilmClub Concierge - Movie Trivia and FAQs

Content covers:
- Common movie FAQs including director information
- Oscar records and box office records
- Popular movie recommendations by genre
- Famous actors and their notable works
- Movie directors and their masterpieces

### Article 2: FilmClub Collection Management Guide

Content covers:
- How to add movies to the FilmClub collection
- Duplicate prevention guidelines
- How to handle requests where a movie title is missing
- Celebratory response guidelines for new additions

### Article 3: FilmClub - Famous Movie Directors

Content covers detailed profiles of 10 major directors including:
- Francis Ford Coppola
- Martin Scorsese
- Christopher Nolan
- Steven Spielberg
- Quentin Tarantino
- Stanley Kubrick
- Alfred Hitchcock
- Ridley Scott
- David Fincher
- Bong Joon-ho

### Article 4: FilmClub - Famous Actors and Their Works

Content covers detailed profiles of 8 major actors including:
- Al Pacino
- Robert De Niro
- Leonardo DiCaprio
- Meryl Streep
- Jack Nicholson
- Morgan Freeman
- Cate Blanchett
- Denzel Washington

### Article 5: FilmClub - Movie Recommendations by Genre

Content covers curated movie lists for 8 genres:
- Drama
- Thriller
- Science Fiction
- Comedy
- Action
- Romance
- Horror
- Animation

Also includes similar movie recommendations for Interstellar and The Godfather.

### Article 6: FilmClub - Movie Trivia and Fun Facts

Content covers:
- Oscar records and firsts
- Box office records
- Movie firsts in cinema history
- Famous movie quotes and their films
- Interesting behind the scenes facts
- Movie sequels and franchises

---

## Permissions Configuration

### Einstein Agent User Permissions

The following permissions were configured for the Einstein Agent User:

Permission Set: FilmClub_Concierge_Agent Permissions (auto-generated)
- Movie_T__c Object: Read, Create, Edit enabled
- Movie_T__c Fields: Read and Edit enabled for all fields
- External Credential Principal Access: OMDb External - OMDB Principal enabled
- System Permission: Allow View Knowledge enabled

---

## Final Test Results

| Test | Query | Result |
|------|-------|--------|
| 1 | I watched an amazing movie last night! | Passed - Agent asked for movie title |
| 2 | Who directed The Godfather? | Passed - Detailed accurate response |
| 3 | Can you suggest some good movies for me? | Passed - Asked for preferences |
| 4 | Tell me about Al Pacino | Passed - Detailed filmography and facts |
| 5 | I am in the mood for a thriller | Passed - Curated list of thrillers |
| 6 | I loved a movie I saw recently | Passed - Asked for movie title |
| 7 | What are some good movies like Interstellar? | Passed - Relevant recommendations |
| 8 | Tell me some fun facts about The Godfather | Passed - 10 detailed fun facts |
| 9 | Tell me about Christopher Nolan | Passed - Detailed director profile |

## Achievements Summary

| What | Status |
|---|---|
| Built and debugged an Auto-Launched Flow | Done |
| Created a fully functional Agentforce Agent | Done |
| Configured 5 topics with detailed instructions | Done |
| Created 6 Knowledge Articles for grounding | Done |
| Fixed content sensitivity filter issues | Done |
| Achieved 9/9 passing tests | Done |

---

## Known Limitations and Inconsistent Behaviour

### Non-Deterministic Response Grounding

One important limitation observed after deployment is that agent responses are not always consistent. The same question can produce a correct detailed response one time and a generic fallback response the next time, even without any changes to the configuration.

This behaviour was specifically observed with:
- "Who directed The Godfather?"
- "Tell me about Al Pacino"
- Questions about crime and mature themed films

### Root Causes

1. The Knowledge Article retrieval is probabilistic. The Dynamic Retriever does not always locate the same article chunk for the same query. Sometimes it finds relevant content and grounds the response successfully. Other times it returns no match and the grounding system rejects the response.

2. The content sensitivity filter is inconsistent. Movies and actors associated with crime or mature themes are flagged unpredictably. The same query passes on one attempt and fails on the next.

3. Response generation varies. The AI generates slightly different responses each time. Some variations pass the grounding evaluation and some do not.

### Why This Is More Pronounced in a Free Developer Edition Org

- Data Cloud is not available, which would provide more reliable semantic search and grounding infrastructure.
- The standard Dynamic Retriever cannot be fine-tuned or replaced in a free org.
- The grounding architecture is significantly more reliable in paid Salesforce orgs with Data Cloud enabled.

### Recommendations for Improving Consistency

1. Add more specific and detailed content to Knowledge Articles. The more granular the article content, the higher the probability the retriever finds a relevant match.
2. Use more specific queries. For example, "Tell me about Francis Ford Coppola" tends to be more consistently grounded than "Who directed The Godfather?"
3. For production deployments, upgrading to a Salesforce org with Data Cloud enabled will significantly improve grounding reliability and response consistency.

---

## Key Lessons Learned

1. Agentforce in a free Developer Edition org has significant limitations with external API callouts from agent-triggered flows due to Einstein Agent User permission restrictions.

2. The response grounding system is fundamental to Agentforce and cannot be bypassed. All agent responses must be backed by a verifiable data source such as Salesforce Knowledge Articles.

3. Salesforce Knowledge Articles serve as the grounding foundation for Agentforce agents without Data Cloud. Well-structured articles with comprehensive content are essential for consistent agent behaviour.

4. Topic Classification Descriptions are critical for correct topic selection. Vague descriptions cause the agent to select wrong topics. Specific and explicit descriptions produce consistent behaviour.

5. Content sensitivity filters in Agentforce block responses about movies with crime, violence or mature themes even in educational contexts. Explicit topic instructions are required to allow discussion of these cinematic works.

6. The Agent User permission configuration is more complex than standard Salesforce user permissions. Both the manually created Permission Set and the auto-generated Permission Set must be correctly configured.

7. Debug mode in Flow Builder does not fully replicate agent-triggered flow execution. A flow working in Debug does not guarantee it will work when triggered by an agent.

8. Salesforce Classic is required to edit and publish Knowledge Articles when Lightning Experience does not expose the Body field as editable through standard inline editing.

---

## Tools and Features Used

- Agentforce Agent Builder
- Einstein Generative AI Topic Generation
- Salesforce Flow Builder (Auto-Launched Flow)
- Salesforce Knowledge
- Prompt Builder
- External Services (OMDBServiceV2)
- Named Credentials and External Credentials
- Permission Sets
- Salesforce Classic (for Knowledge Article editing)

---

## Date of Implementation
March 2026

---

## Author
Suhas Kadhane
Salesforce Administrator and Developer
