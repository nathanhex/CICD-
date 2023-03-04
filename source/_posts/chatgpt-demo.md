---
title: openai gpt-3.5-turbo新模型java demo
categories:
  - [chatgpt]
tags:
    - openai
    - chatgpt
    - java
    - gpt-3.5-turbo
date: 2023-03-04
---
#  openai gpt-3.5-turbo新模型java demo
> openai 新增了gpt-3.5-turbo模型，该模型可以有chatgpt网页一样的效果，可以记住上下文信息。官方提供了python和nodejs的sdk，我基于java的sdk写了一个demo
## 代码依赖：
```xml
    <dependencies>
        <!--
        openai的一个开源的java sdk库，git地址是：https://github.com/TheoKanning/openai-java/tree/main 
        -->
        <dependency>
            <groupId>com.theokanning.openai-gpt3-java</groupId>
            <artifactId>service</artifactId>
            <version>0.11.0</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.32</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
            <version>1.18.16</version>
        </dependency>
    </dependencies>
```
## 代码示例：
```java
package org.yun.chatgpt.demo;

import com.theokanning.openai.completion.chat.ChatCompletionRequest;
import com.theokanning.openai.completion.chat.ChatMessage;
import com.theokanning.openai.service.OpenAiService;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashMap;
import java.util.Map;
import java.util.Scanner;

import lombok.extern.slf4j.Slf4j;

@Slf4j
public class GPT3Demo {
    private final OpenAiService openAiService;
    /**
     * 存放一个session的一组ChatMessage，
     */
    private final Map<String, ArrayList<ChatMessage>> sessionMap = new HashMap<>();

    /**
     * 控制对话的次数，当10轮对话20个chatmessage对象后强制重新开始一轮对话，
     */
    private int maxRounds = 20;

    public GPT3Demo(String token) {
        this.openAiService = new OpenAiService(token);
    }

    public String chat(String sessionId, String question) {
        log.debug("ChatGPTBoot.chat sessionId: {};que:{}", sessionId, question);
        ArrayList<ChatMessage> chatMessages = sessionMap.get(sessionId);
        //当对话10轮后强制重新开始对话
        if (chatMessages == null || chatMessages.size() > maxRounds) {
            chatMessages = new ArrayList<>();
            sessionMap.put(sessionId, chatMessages);
        }
        chatMessages.add(new ChatMessage("user", question));
        if (log.isDebugEnabled()) {
            log.debug("ChatGPTBoot.chat sessionId={};chatMessages:{}", sessionId,
                    chatMessages.toArray());
        }
        //参数数目请查看readme
        ChatCompletionRequest completionRequest =
                ChatCompletionRequest.builder()
                        .messages(chatMessages)
                        .model("gpt-3.5-turbo")
                        .temperature(0.7)
                        .maxTokens(2500)
                        .topP(1.0)
                        .frequencyPenalty(0.0)
                        .stop(Arrays.asList("\n", " Human:", " AI:"))
                        .presencePenalty(0.0)
                        .build();


        ArrayList<String> c = new ArrayList<>();
        ArrayList<ChatMessage> c2 = new ArrayList<>();
        openAiService.createChatCompletion(completionRequest).getChoices().forEach(v -> {
            ChatMessage message = v.getMessage();
            c2.add(message);
            c.add(message.getContent());
        });
        chatMessages.addAll(c2);
        return String.format("%s", c.toArray());
    }
    public static void main(String[] args) {

        GPT3Demo gpt3Demo = new GPT3Demo("your token");
        Scanner sco = new Scanner(System.in);
        System.out.println("your question:");
        String question;
        while (sco.hasNext()) {
            question = sco.next();
            if ("quit".equals(question) || "exit".equals(question)) {
                break;
            }
            String a = gpt3Demo.chat("1", question);
            System.out.println("boot:[" + a + "]");
        }
        sco.close();
    }
}
```
## openai参数说明：
+ 【max_tokens】integer 可选 默认值16
  完成时要生成的最大token数量。

提示的token计数加上max_tokens不能超过模型的上下文长度。大多数模型的上下文长度为2048个token（最新模型除外，支持4096个）。
+ 【temperature】number 可选 默认值1
  使用什么样的采样温度，介于0和2之间。较高的值（如0.8）将使输出更加随机，而较低的值（例如0.2）将使其更加集中和确定。

通常建议更改它或top_p，但不能同时更改两者。
+ 【top_p】number 可选 默认值1
  一种用温度采样的替代品，称为核采样，其中模型考虑了具有top_p概率质量的token的结果。因此，0.1意味着只考虑包含前10%概率质量的token。

通常建议改变它或temperature，但不能同时更改两者。

+ 【presence_penalty】number 可选 默认值0
  取值范围：-2.0~2.0。
  正值根据新token到目前为止是否出现在文本中来惩罚它们，这增加了模型谈论新主题的可能性。

+ 【frequency_penalty】number 可选 默认值0
  取值范围：-2.0~2.0。
  正值根据迄今为止文本中的现有频率惩罚新token，从而降低了模型逐字重复同一行的可能性。

+ 【best_of】integer 可选 默认值1
  在服务器端生成best_of个完成，并返回“最佳”（每个token的日志概率最高）。结果无法流式传输。

与n一起使用时，best_of控制候选完成的数量，n指定要返回的数量–best_of必须大于n。

注意：由于此参数会生成许多完成，因此它可以快速消耗token配额。小心使用并确保您对max_tokens和stop进行了合理的设置。

+ 【logit_bias】map 可选 默认值null
  修改完成时出现指定token的可能性。

接受将token（由其在GPT token生成器中的token ID指定）映射到从-100到100的相关偏差值的json对象。
您可以使用此token工具（适用于GPT-2和GPT-3）将文本转换为token ID。在数学上，偏差在采样之前被添加到模型生成的逻辑中。确切的效果因模型而异，但-1和1之间的值应该会降低或增加选择的可能性；-100或100这样的值应该导致相关token的禁止或独占选择。

例如，可以传递｛“50256”：-100｝以防止生成<|endoftext|>的token。
## 完整的代码
[https://github.com/nathanhex/chatgpt-demo](https://github.com/nathanhex/chatgpt-demo)
**完**