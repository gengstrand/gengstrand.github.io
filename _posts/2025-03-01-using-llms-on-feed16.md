---
layout: post
status: publish
permalink: /software/llm/migration/java/typescript
published: true
title: Using Large Language Models to Migrate a News Feed Microservice
author:
  display_name: glenn
  login: glenn
  url: https://glennengstrand.info
author_login: glenn
author_url: https://glennengstrand.info
date: '2025-03-01 09:01:38 -0700'
date_gmt: '2025-03-01 13:01:38 -0700'
categories:
- Technology
- Artificial Intelligence
- Large Language Models
tags:
- SpringBoot
- TypeScript
- Nest
- Llama
- Copilot
comments: []
---
<script async src="https://www.googletagmanager.com/gtag/js?id=G-JMTSMFHZKD"></script>
<script src="/assets/ga.js"></script>

Last year, I wrote a piece for InfoQ on [Experimenting with LLMs for Developer Productivity](https://www.infoq.com/articles/llm-productivity-experiment/). This year, I decided to expand on that exploration by using LLMs to add another implementation of the news feed service to my [learning repository](https://github.com/gengstrand/clojure-news-feed). In this repository, I implement the same feature-identical rudimentary news feed microservice in various tech stacks and programming languages. I had heard a lot of good things about [Nest.js](https://nestjs.com/) so I decided that it was time to introduce Typescript on Nest.js using TypeORM as [feed 16](https://github.com/gengstrand/clojure-news-feed/tree/master/server/feed16). Nest uses Express, but it introduces a quasi-dependency injection way of organizing the code that is reminiscent of Spring Boot.

## Initial Migration with Llama 3.3

I started this migration adventure by running the following program on my already coded [Java on Spring Boot](https://github.com/gengstrand/clojure-news-feed/tree/master/server/feed8) (a.k.a. Feed 8) project using [Llama 3.3](https://ollama.com/library/llama3.3) as the model.

```python
import os
import sys
from pathlib import Path
from langchain_ollama.llms import OllamaLLM

def main(model: str, inpath: str, outpath: str):
    llm = OllamaLLM(model=model)
    for root, dirs, files in os.walk(inpath):
        out_dir = Path(root.replace(inpath, outpath))
        out_dir.mkdir(parents=True, exist_ok=True)
        for name in files:
            np = name.split('.')
            ft = np[-1]
            if ft == 'java':
                fni = root + '/' + name
                with open(fni, 'r') as f:
                    java_code = f.read()
                    prompt = "Migrate the following code which is written in Java on Spring Boot with Spring Data to code that is written in Typescript on NestJS with TypeORM.\n\n" + java_code
                    fno = fni.replace(inpath, outpath).replace('.java', '.ts')
                    with open(fno, 'w') as o:
                        o.writelines(llm.invoke(prompt))

if __name__ == "__main__":
    if len(sys.argv) < 4:
        print("usage: python code_migrator.py model root_java_repo root_ts_repo")
        sys.exit(-1)
    main(sys.argv[1], sys.argv[2], sys.argv[3])
```

I don't have a GPU on my laptop, so I spun up an "a2-highgpu-1g" VM with an NVIDIA A100 40GB and an 80 GB SCSI disk running machine learning on Linux with CUDA on GCP. If memory serves, this costs about $4 per hour. This program took about a half hour to run.

### Observations on how Llama Performed

Most of the fixes I had to make were due to these shortcomings.

* Llama did not know that the default HTTP status code for the POST methods in Nest.js is 201 whereas the default HTTP status code for POST methods in Spring Boot is 200. I had to override that.
* It almost generated the TypeORM entities correctly, but it did not really understand how to inject them into the service using Nest.js. The same is true for Redis and ElasticSearch. It also kind of gave up on the Cassandra code.
* It understood the need to use Jest in the unit tests and verify that the TypeORM repository methods were being called, but it did not understand that it also had to mock the Redis caching methods. 
* It generated the TypeORM entities, controller requests, and response types but nothing for the service layer. Instead, the service depended on the controller types, resulting in a circular dependency and not being the best approach for the caching layer. I wrote similar classes for the service and code that converted between the related classes in the various layers. 
* Was that really necessary? Since the tsc tool is a transpiler, not a real compiler, I suspect circular dependencies are not a show-stopper. It still leads to code that is harder to understand. The "Feed 8" project also did not have separate POJO classes for service and controller, but the idiomatic way to organize Spring Boot projects avoids the circular dependency issue. Should Llama have been able to foresee this? Bob Martin talks about maintaining a clear separation of concerns in his controversial book entitled *Clean Architecture* but I never mentioned anything about improving code quality in the prompt. Most competent human engineers look for ways to enhance code quality with any change.

## Fixing the Code with GitHub Copilot in VS Code

I resisted Copilot during my evaluation back in 2024 but decided to go all in with it this time since it is free now.

I initialized the project with this [typescript-starter](https://github.com/nestjs/typescript-starter/tree/master) template. Then, I replaced the generic app code with what Llama produced. I renamed the files and moved them to a more Nest-appropriate folder structure, removed the Llama commentary, and did some more code refactoring to be more idiomatic of Nest. Then, I fired up [VS Code with the Copilot plugin](https://code.visualstudio.com/docs/copilot/overview) and got to work fixing everything that was broken.

Copilot's statement completion on steroids capability was very similar to what I experienced with Amazon Code Whisperer. It sure does save on typing. It is usually accurate and gets better the longer you use it. However, do not let that lull you into a false sense of contentment. It can and does make mistakes that can be pretty stupid. One time, it generated a block of code where one line in the middle was just a variable name and a semicolon. It didn't do anything. I completely missed that. It was the linter that finally caught it. Review fatigue is very real.

The inline chat capability was more hit-or-miss. It usually does okay if you ask it to write a simple function describing what that function should do. It is not so good when you ask it to fix a bug or resolve a compiler issue. Like Llama, Copilot also struggled with Nest-style dependency injection and came up short there.

## Are LLMs Worth It?

It took me a week to take the Llama-generated code and make it Nest idiomatic, feature complete, and pass both unit and integration tests. That is 15 files, totaling 886 lines of code. I did not keep track of my time on this, and I was busy with other projects during that timeframe. I did not work on this full-time. The previous feature identical implementation, Feed 15, was Kotlin on Spring Web Flux, which initially took me about a month to code. I also did not work on Feed 15 full-time. That implementation was 17 files and 1017 lines coded by hand.

A simple back-of-the-envelope calculation yields that using these LLMs tripled my productivity. That is most likely a conservative estimate. Remember that most coders have other responsibilities that are not included in this evaluation.

## Looking Back

It's been nine months since that InfoQ article was published. What has changed in this space since then? A lot but also very little. Regarding the Gartner Hype Cycle, we are still heading towards the peak of inflated expectations, but I do catch the occasional article hinting that there might be a trough of disillusionment somewhere in the future. The Open AI folks have decided to make progress on their goal of achieving AGI by redefining AGI as something easier to achieve. The context windows have gotten considerably larger. Many LLMs are now multimodal. With some notable exceptions, building LLMs has become more efficient, leading to lower costs and lowering the barriers to entry. The variety and quality of tools and technologies, both open-source and cloud-based, make it easier to distill, fine-tune, or RAG your LLMs. A majority of companies now incorporate LLMs into their workflows or offerings. Not much has changed in the humble world of LLM fueled code generation. Statement completion is still the killer app. LLMs continue to disrupt Q&A platforms. LLMs still don't understand annotation (a.k.a. decorators in Typescript) based dependency injection.