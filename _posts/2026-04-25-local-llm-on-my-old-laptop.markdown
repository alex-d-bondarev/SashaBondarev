---  
title: Local LLM on my old Laptop  
toc: true  
date: 2026-04-25  
last_modified_at: 2026-04-25  
categories: [ LLM, ollama, OpenCode, gemma4 ]
---  

A few months ago, I tried to run LLM locally. It worked on my Raspberry Pi with 4GB RAM, but very slowly.
And a small RAM limited the models I could run.

Then Google released Gemma 4. So I decided to test if it can run locally on my old MacBook laptop
with 16GB RAM and 3.1 GHz Dual-Core Intel CPU as a server via [Ollama](https://ollama.com/),
use it via [OpenCode](https://opencode.ai/) on my other laptop as a client and if it's better than other models?

Of course, “better” is not a very clear word. I actually mean that it can help me with my learning, and that prompts
can be processed while I make a cup of tea or coffee. Also, I cannot rely on LLM tests since
[LLM tests can be hacked](https://rdi.berkeley.edu/blog/trustworthy-benchmarks-cont/).
So I need to test them the way I use LLMs.

TL;DR Gemma 4 is the fastest helpful LLM you can run locally on an old machine.
Below, you can find how I came to this conclusion.

## Setup

I assume that you know how to set up environment variables and run commands in the terminal.
If not, there are a lot of great articles and courses that can describe this much better than me.

### On my old laptop (Server)

1. I noted my laptop local Wi-Fi IP via `ifconfig | grep "inet 192"`
2. added `OLLAMA_HOST="0.0.0.0:12345"` environment variable so that ollama can receive prompts from
   other devices on my Wi-Fi.
3. increased [default ollama context window](https://docs.ollama.com/context-length) from 4K to 32K
   by adding `OLLAMA_CONTEXT_LENGTH=32768` environment variable. P.S. I later reduced it to `16384`.
4. started the ollama process via `ollama serve`
5. pulled Gemma 4 via `ollama pull gemma4:e2b-it-q4_K_M`

### On my newer laptop (Client)

1. I installed [OpenCode](https://opencode.ai/)
2. created `~/.config/opencode/opencode.json` file with the following content:
   ```json  
    {
      "$schema":"https://opencode.ai/config.json",
      "provider":{
        "localOllama":{
          "npm":"@ai-sdk/openai-compatible",
          "name":"Ollama (local)",
          "options":{
            "baseURL":"http://<laptop_ip>:12345/v1"
          },
          "models":{
            "gemma4:e2b-it-q4_K_M":{
              "name":"Gemma 4 (Local)"
            }
          }
        }
      }
    }
   ```
3. launched an `opencode` session, typed `hi!` and received: `Hi there! How can I help you today?` response.

## A few small head bumps

Yeah, it worked! Now what?

Thus, I have generated a pretty broken project that I plan to use to test several tools and approaches.
Meet [Inventory Control Utility (ICU)](https://github.com/alex-d-bondarev/demo-web-app).
I have also came up with 4 prompts against it:

1. Load entire project and ask:
    ```  
    You are given a legacy project "ICU". You have to understand how it works.   
    Your goal is to summarise issues that you have found and share a step-by-step plan to fix them.  
    ```
2. Load front-end folder and ask:
    ```  
    I have some basic knowledge about HTML, CSS and JavaScript.   
    But I do not know how NodeJS works.   
    Please explain to me how `frontend` is implemented in this project.  
    ```
3. Load GitHub workflow and ask:
    ```  
    You are an experienced software architect that cares about software best practicies, such as, 
    but not limited to scalability, testability, stability, maintainability and readability.  
    I need you to check the `.github/workflows/build.yml` CI pipeline.  
    What are your impressions about it? Would you change anything?
    ```  
4. Pretend to be an SDET and analyze a specific step of the GitHub workflow:
    ```  
    You are an experienced software developer in test that cares about software best practicies, such as,   
    but not limited to tests stability, readability and maintainability.  
    I need you to check `Verify API Endpoints` step in the `.github/workflows/build.yml` pipeline.       
    Is it acceptable or would you change anything?    
    ```

I was able to test those prompts with a free GPT-4.1, and all 4 took around 6.5 minutes.
That sounded like a good start. So I found a few models that should fit my laptop's RAM.

First, I tried `deepseek-coder-v2`, and it failed since it does not support LLM tools. Next, I tried `devstral:24b`,
which quickly used the entire RAM on my old laptop, started to process something,
and 1.5 hours later raised a 500 error.

This way I learned to look for the `Tools` tag in the [ollama search](https://ollama.com/search)
and that I should not use models that have a size larger than half my RAM. I have also realized that it would take
eternity to check all 4 prompts. And you can easily multiply this by 2, given how accurate I am at making estimates.

## A new test is born

I decided to limit my overcomplicated test to the following:

1. Find LLM models that have the `Tools` tag in the [ollama search](https://ollama.com/search).
2. Filter by size of 8GB or less (half of my old laptop RAM).
3. Use only a single prompt that was about GitHub workflow, which took GPT-4.1 just 16 seconds to process.

This helped me to come up with a list of models:

| Name            | Model ID                                                                                | Model Size |  
|-----------------|-----------------------------------------------------------------------------------------|------------|  
| Cogito          | [cogito:3b-v1-preview-llama-fp16](https://ollama.com/library/cogito/tags)               | 7.2GB      |  
| Gemma 4         | [gemma4:e2b-it-q4_K_M](https://ollama.com/library/gemma4)                               | 7.2GB      |  
| Granite 3       | [granite3.3:8b](https://ollama.com/library/granite3.3/tags)                             | 4.9GB      |
| Hermes 3        | [hermes3:8b-llama3.1-q6_K](https://ollama.com/library/granite3.3/tags)                  | 6.6GB      |
| Llama 3.2       | [llama3.2:3b-instruct-fp16](https://ollama.com/library/llama3.2/tags)                   | 6.4GB      |
| Ministral 3     | [ministral-3:3b-instruct-2512-fp16](https://ollama.com/library/ministral-3/tags)        | 7.7GB      |
| Mistral Nemo    | [mistral-nemo:12b-instruct-2407-q4_K_M](https://ollama.com/library/granite3.1-moe/tags) | 7.5GB      |
| Nemotron 3 Nano | [nemotron-3-nano:4b-bf16](https://ollama.com/library/nemotron-3-nano/tags)              | 5.1GB      |
| Phi 4 Mini      | [phi4-mini:3.8b-fp16](https://ollama.com/library/phi4-mini/tags)                        | 7.7GB      |
| Qwen 2.5 Coder  | [qwen2.5-coder:14b-instruct-q3_K_L](https://ollama.com/library/qwen2.5-coder/tags)      | 7.9GB      |
| Qwen 3.5        | [qwen3.5:9b-q4_K_M](https://ollama.com/library/qwen3.5/tags)                            | 6.6GB      |
| RNJ 1           | [rnj-1:8b-instruct-q4_K_M](https://ollama.com/library/rnj-1/tags)                       | 5.1GB      |
| Smollm 2        | [smollm2:1.7b-instruct-fp16](https://ollama.com/library/smollm2/tags)                   | 3.4GB      |
| Starcoder 2     | [starcoder2:7b-q8_0](https://ollama.com/library/starcoder2/tags)                        | 7.6GB      |

I thought it would be a walk in the park… and ended up spending almost 5 days running this test. Why so long?
Cos prompts would take up to 3 hours to process, and I could not just sit in front of the laptop and start a new prompts
immediately after the previous one finishes. I think it is better to split the results into 2 categories:

## Failure results

You can see more error details if you click on the errors

| Model Name     | Error                                                                                                                                                  | RAM used | Time before it failed |
|----------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|----------|-----------------------|
| Cogito         | [Failed to read file](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Cogito/GitHub_workflow.md)                        | 12GB     | 3h 48m                |
| Granite 3      | [Failed to read file](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Granite_3_3/GitHub_workflow.md)                   | 12GB     | 2h 42m                |
| Hermes 3       | [It could not finish a read file step](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Hermes_3/GitHub_workflow.md)     | 9.5GB    | 1h 18m                |
| Llama 3.2      | [Error 500 in ollama logs](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Llama_3_2/GitHub_workflow.md)                |          | almost immediately    |
| Mistral Nemo   | [It could not finish a read file step](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Mistral_Nemo/GitHub_workflow.md) | 9.5GB    | 3h 9m                 |
| Phi 4 Mini     | [Probably(?) failed to use tools](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Phi_4_Mini/GitHub_workflow.md)        | 9.7GB    | 1h 31m                |
| Qwen 2.5 Coder | [Probably(?) failed to use tools](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Qwen_2_5_Coder/GitHub_workflow.md)    | 9.6GB    | 3h 43m                |
| RNJ 1          | [Error 500 in ollama logs](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/RJU_1/GitHub_workflow.md)                    |          | almost immediately    |
| Smollm 2       | [Failed to load skills](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Smollm_2/GitHub_workflow.md)                    |          | almost immediately    |
| Starcoder 2    | [Does not support tools](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Starcoder_2/GitHub_workflow.md)                |          | almost immediately    |

Yeah, looks like the tools tag on the Ollama website is outdated for some models.

P.S. I did not care much why these models failed, since I had enough models that did not.

## Successful results

You can see more error details if you click on the summaries

| Model Name      | Summary                                                                                                                                                                                                                                                    | RAM used | Response time |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|---------------|
| GPT 4.1         | [Caught all main points. Repeated/paraphrased some of them for no reason. It has either analysed more than 1 file or hallucinated some points.](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/GPT_4_1/GitHub_workflow.md) | Cloud    | 16s           |
| Gemma 4         | [Caught all main points. Concise response.](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Gemma%204/GitHub_workflow.md)                                                                                                   | 9.3GB    | 21m           |
| Ministral 3     | [It focused on low priority items and made suggestions that I consider as bad practice. I'm definitelly not using it!](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Ministral_3/GitHub_workflow.md)                      | 13GB     | 2h 7m         |
| Nemotron 3 Nano | [I don't think it caught main problems. Average response.](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Nemotron_3_Nano/GitHub_workflow.md)                                                                              | 14GB     | 53m 4s        |
| Qwen 3.5        | [It did not suggest to replace curl checks with proper tests. The rest is ok.](https://github.com/alex-d-bondarev/llm-experiments/blob/main/local_llm/results/Qwen_3_5/GitHub_workflow.md)                                                                 | 12GB     | 1h 14m        |

## Summary

I need a better machine to run LLMs locally or pay the premium for LLMs in cloud! But still...

1. `Gemma 4` was good enough and gave the best answer compared to other local LLMs. It was also the fastest.
2. I would give `Qwen 3.5` a second place. It's response was not bad, but was 3.5 slower than `Gemma 4`.
3. I did not like `Ministral 3` and `Nemotron 3 Nano` responses. I would not recommend them for coding tasks.
4. The other models failed for multiple reasons.

So, I suggest to use `Gemma 4` if you have some old hardware comparable or more capable than MacBook from 2015.

Thank you for reading!
