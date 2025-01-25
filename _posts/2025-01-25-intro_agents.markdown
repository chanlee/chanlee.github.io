---
layout: post
title: '에이전트와 리액트(ReAct) 소개'
date: 2025-01-25 01:20:00 +0900
categories: ai agents smolagents huggingface
published: true
---

원문 :

-   [Introduction to Agents][origin]{:target="\_blank"}
-   [How do multi-step agents work?](https://huggingface.co/docs/smolagents/conceptual_guides/react){:target="\_blank"}

## 🤔 에이전트란 무엇인가요?

AI를 사용하는 모든 효율적인 시스템은 LLM에게 실제 세계에 대한 어떤 형태의 접근을 제공해야 합니다: 예를 들어 외부 정보를 얻기 위해 검색 도구를 호출하거나, 작업을 해결하기 위해 특정 프로그램에 작용할 수 있는 능력이 필요합니다. 다시 말해서, LLM은 행위성(agency)을 가져야 합니다. 에이전트 프로그램은 LLM이 외부 세계로 나아가는 관문(gateway)입니다.

> AI 에이전트는 LLM 출력이 워크플로우를 제어하는 프로그램입니다.

LLM을 활용하는 모든 시스템은 LLM의 출력을 코드에 통합할 것입니다. 코드 워크플로우에 대한 LLM 입력의 영향력이 바로 시스템에서 LLM의 에이전시 수준입니다.

이 정의에서 "에이전트"는 이산적인 0 또는 1의 정의가 아닙니다. 대신, "에이전시"는 워크플로우에서 LLM에 더 많거나 적은 권한을 부여함에 따라 연속적인 스펙트럼 상에서 발전합니다.

아래 표에서 시스템 간 에이전시가 어떻게 다양할 수 있는지 확인해보세요:

| 에이전시 수준 | 설명                                                                        | 명명                        | 예시 패턴                                          |
| ------------- | --------------------------------------------------------------------------- | --------------------------- | -------------------------------------------------- |
| ☆☆☆           | LLM 출력은 프로그램 흐름에 영향을 미치지 않습니다.                          | 단순 프로세서               | `process_llm_output(llm_response)`                 |
| ★☆☆           | LLM 출력이 if/else 스위치를 결정합니다.                                     | 라우터                      | `if llm_decision(): path_a() else: path_b()`       |
| ★★☆           | LLM 출력이 함수 실행을 결정합니다.                                          | 도구(tool) 호출             | `run_function(llm_chosen_tool, llm_chosen_args)`   |
| ★★★           | LLM 출력이 반복과 프로그램 진행을 제어합니다                                | 다단계(Multi-step) 에이전트 | `while llm_should_continue(): execute_next_step()` |
| ★★★           | 하나의 에이전틱 워크플로우가 다른 에이전틱 워크플로우를 시작할 수 있습니다. | 다중(Multi) Agent           | `if llm_trigger(): execute_agent()`                |

다단계(multi-step) 에이전트의 코드 구조는 다음과 같습니다:

```python
memory = [user_defined_task]
while llm_should_continue(memory): # this loop is the multi-step part
    action = llm_get_next_action(memory) # this is the tool-calling part
    observations = execute_action(action)
    memory += [action, observations]
```

이 에이전트 시스템은 루프(loop)로 실행되며, 각 단계에서 새로운 동작을 수행합니다(이러한 동작에는 단순한 함수인 미리 정의된 도구들을 호출하는 것이 포함될 수 있습니다). 이는 관찰을 통해 주어진 작업을 해결하기에 충분한 상태에 도달했음이 확인될 때까지 계속됩니다. 다음은 다단계 에이전트가 간단한 수학 문제를 해결하는 방법의 예시입니다:

![simple math question](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/Agent_ManimCE.gif)

## ✅ 에이전트를 언제 사용하는가? / ⛔ 언제는 피해야 하는가?

앱의 워크플로우를 결정하기 위해 LLM이 필요할 때 에이전트가 유용합니다. 하지만 대개는 과도한 경우가 많습니다. 핵심 질문은 이것입니다: 현재 작업을 효율적으로 해결하기 위해 워크플로우의 유연성이 정말 필요한가요? 미리 정해진 워크플로우가 자주 부족하다면, 더 많은 유연성이 필요하다는 의미입니다. 예를 들어보겠습니다. 서핑 여행 웹사이트에서 고객 문의를 처리하는 앱을 만든다고 가정해봅시다.

사용자의 선택에 따라 문의가 두 가지 유형 중 하나에 속할 것이라는 것을 미리 알 수 있고, 각각의 경우에 대해 미리 정의된 워크플로우가 있습니다.

1. 여행 정보가 필요하신가요? ⇒ 지식 베이스를 검색할 수 있는 검색창을 제공합니다.
2. 영업팀과 상담하고 싶으신가요? ⇒ 문의 양식을 작성하도록 합니다.

이러한 결정적 워크플로우가 모든 문의에 적합하다면, 모든 것을 코드로 작성하면 됩니다! 이렇게 하면 예측 불가능한 LLM이 워크플로우에 개입하면서 발생할 수 있는 오류의 위험이 없는 100% 신뢰할 수 있는 시스템을 얻을 수 있습니다. 단순성과 안정성을 위해 에이전트 동작을 사용하지 않는 방향으로 설계하는 것이 좋습니다.

하지만 워크플로우를 미리 그렇게 명확하게 결정할 수 없다면 어떨까요?

예를 들어, 사용자가 이런 문의를 한다고 합시다. "월요일에 갈 수 있는데 여권을 잊어서 수요일로 미뤄질 수 있을 것 같아요. 취소 보험에 가입한 상태로 화요일 아침에 저와 제 장비를 데리고 서핑을 갈 수 있을까요?" 이 질문은 여러 요소에 달려있으며, 앞서 언급한 미리 정해진 기준들로는 이 요청을 처리하기에 충분하지 않을 것입니다.

미리 정해진 워크플로우가 자주 부족하다면, 더 많은 유연성이 필요하다는 의미입니다.

이럴 때 에이전트 설정이 도움이 됩니다.

위 예시의 경우, 날씨 예보를 위한 날씨 API, 이동 거리 계산을 위한 구글 맵스 API, 직원 가용성 대시보드, 그리고 지식 베이스의 RAG 시스템에 접근할 수 있는 다단계 에이전트를 만들 수 있습니다.

최근까지 컴퓨터 프로그램은 미리 정해진 워크플로우에 제한되어 있었고, if/else 분기문을 쌓아가며 복잡성을 처리하려 했습니다. "이 숫자들의 합계 계산" 또는 "이 그래프에서 최단 경로 찾기"와 같은 매우 제한적인 작업에만 집중했죠. 하지만 실제로 위의 여행 예시처럼 대부분의 현실 세계의 작업은 미리 정해진 워크플로우에 들어맞지 않습니다. 에이전트 시스템은 프로그램이 실제 세계의 다양한 작업을 처리할 수 있도록 새로운 가능성을 열어줍니다!

## 왜 smolagents 인가?

체인이나 라우터와 같은 일부 저수준 에이전트 사용 사례의 경우, 모든 코드를 직접 작성할 수 있습니다. 이렇게 하면 시스템을 더 잘 제어하고 이해할 수 있어 훨씬 더 효과적입니다.

하지만 LLM이 함수를 호출하도록 하거나("tool calling") LLM이 while 루프를 실행하도록 하는("multi-step agent") 것과 같은 더 복잡한 동작을 구현하기 시작하면 몇 가지 추상화가 필요해집니다:

-   도구 호출의 경우, 에이전트의 출력을 구문 분석해야 하므로 출력은 "생각: 'get_weather' 도구를 호출해야 합니다. 행동: get_weather(Paris)"와 같은 미리 정의된 형식이 필요합니다. 이는 미리 정의된 함수로 구문 분석되며, LLM에 제공되는 시스템 프롬프트에서 이 형식을 명시해야 합니다.
-   LLM 출력이 루프를 결정하는 다단계 에이전트의 경우, 이전 루프 반복에서 발생한 결과에 따라 LLM에 다른 프롬프트를 제공해야 하므로 일종의 메모리가 필요합니다.

이해가 되시나요? 이 두 가지 예시만으로도 우리에게 필요한 몇 가지 요소들이 드러났습니다:

-   당연히 시스템의 동력이 되는 LLM
-   에이전트가 접근할 수 있는 도구 목록
-   LLM 출력에서 도구 호출을 추출하는 파서
-   파서와 동기화된 시스템 프롬프트
-   메모리

하지만 주의하세요. LLM에게 의사 결정의 여지를 주면 필연적으로 실수가 발생할 것입니다. 따라서 오류 로깅과 재시도 메커니즘이 필요합니다.

이러한 모든 요소들은 제대로 작동하는 시스템을 구축하기 위해 긴밀하게 연결되어야 합니다. 그래서 우리는 이 모든 것들이 함께 작동할 수 있도록 기본 구성 요소를 만들기로 결정했습니다.

## 코드 에이전트 (Code Agents)

멀티스텝 에이전트에서는 각 단계마다 LLM이 외부 도구를 호출하는 형태로 액션을 작성할 수 있습니다. Anthropic, OpenAI 및 기타 여러 기업들이 사용하는 일반적인 형식은 "도구 이름과 사용할 인수를 JSON 형태로 작성한 후, 이를 파싱하여 실행할 도구와 인수를 결정하는" 방식의 다양한 변형입니다.

여러 연구 논문([1](https://huggingface.co/papers/2402.01030){:target="\_blank"}, [2](https://huggingface.co/papers/2411.01747){:target="\_blank"}, [3](https://huggingface.co/papers/2401.00812){:target="\_blank"})에서 도구를 호출하는 LLM을 코드로 작성하는 것이 훨씬 더 효과적이라는 것을 입증했습니다.

이는 단순히 우리가 컴퓨터가 수행하는 작업을 표현하는 가장 좋은 방법으로 프로그래밍 언어를 특별히 설계했기 때문입니다. 만약 JSON 스니펫이 더 나은 표현 방식이었다면, JSON이 최고의 프로그래밍 언어가 되었을 것이고 프로그래밍은 지옥이 되었을 것입니다.

아래 그림은 "[Executable Code Actions Elicit Better LLM Agents](https://huggingface.co/papers/2402.01030){:target="\_blank"}"에서 발췌한 것으로, 코드로 액션을 작성하는 것의 여러 장점을 보여줍니다:

![Executable Code Actions Elicit Better LLM Agents](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/code_vs_json_actions.png)

JSON과 같은 스니펫 대신 코드로 액션을 작성하면 다음과 같은 장점이 있습니다:

-   구성 가능성: 파이썬 함수를 정의하는 것처럼 JSON 액션을 서로 중첩하거나 나중에 재사용할 JSON 액션 세트를 정의하는 것이 가능할까요?
-   객체 관리: JSON에서 generate_image와 같은 액션의 출력을 어떻게 저장할 수 있을까요?
-   일반성: 코드는 컴퓨터가 수행할 수 있는 모든 작업을 간단히 표현할 수 있도록 설계되어 있습니다.
-   LLM 학습 데이터에서의 표현: LLM의 학습 데이터에 이미 많은 양질의 코드 액션이 포함되어 있어 이미 이를 위한 학습이 완료되어 있습니다!

## 다단계 에이전트(multi-step agent)는 어떻게 작동하는가?

ReAct 프레임워크([Yao et al., 2022](https://huggingface.co/papers/2210.03629){:target="\_blank"})는 현재 에이전트를 구축하는 주요 접근 방식입니다.

이 이름은 "Reason(추론)"과 "Act(행동)"이라는 두 단어의 조합에서 유래했습니다. 실제로 이 아키텍처를 따르는 에이전트들은 필요한 만큼의 단계를 거쳐 작업을 해결하는데, 각 단계는 추론 단계와 주어진 작업 해결에 더 가까워지기 위한 도구 호출을 수행하는 행동 단계로 구성됩니다.

smolagents의 모든 에이전트는 ReAct 프레임워크의 추상화인 단일 MultiStepAgent 클래스를 기반으로 합니다.

기본적으로 이 클래스는 다음과 같이 기존 변수와 지식이 에이전트 로그에 통합되는 단계들을 순환적으로 수행합니다:

초기화: 시스템 프롬프트는 SystemPromptStep에 저장되고, 사용자 쿼리는 TaskStep에 기록됩니다.

While 루프(ReAct 루프):

-   agent.write_inner_memory_from_logs()를 사용하여 에이전트 로그를 LLM이 읽을 수 있는 채팅 메시지 목록으로 작성합니다.
-   이 메시지들을 Model 객체에 전송하여 완료된 결과를 얻습니다. 완료된 결과를 구문 분석하여 액션을 얻습니다(ToolCallingAgent의 경우 JSON blob, CodeAgent의 경우 코드 스니펫).
-   액션을 실행하고 결과를 메모리에 기록합니다(ActionStep).
-   각 단계가 끝날 때마다 agent.step_callbacks에 정의된 모든 콜백 함수를 실행합니다.

선택적으로, 계획이 활성화되면 계획을 주기적으로 수정하여 PlanningStep에 저장할 수 있습니다. 여기에는 현재 작업에 대한 사실들을 메모리에 제공하는 것이 포함됩니다.

CodeAgent의 경우, 아래 그림과 같습니다.

![How CodeAgent.run() works](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/smolagents/codeagent_docs.png)

다음은 이것이 작동하는 과정을 비디오로 보여주는 것입니다:

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/Agent_ManimCE.gif)

<br/>

![](https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/blog/open-source-llms-as-agents/ReAct.png)

우리는 두 가지 버전의 에이전트를 구현합니다:

-   [CodeAgent](https://huggingface.co/docs/smolagents/v1.5.0/en/reference/agents#smolagents.CodeAgent){:target="\_blank"}는 선호되는 유형의 에이전트로, 도구 호출을 코드 블록 형태로 생성합니다.

-   [ToolCallingAgent](https://huggingface.co/docs/smolagents/v1.5.0/en/reference/agents#smolagents.ToolCallingAgent){:target="\_blank"}는 일반적인 에이전트 프레임워크처럼 도구 호출을 JSON 형태로 출력합니다. 이 옵션을 포함한 이유는 단계당 하나의 도구 호출만으로도 충분한 특정 상황에서 유용하기 때문입니다. 예를 들어, 웹 브라우징의 경우 페이지의 변화를 확인하기 위해 각 동작 후에 대기해야 합니다.

> 에이전트를 한 번에 실행할 수 있는 옵션도 제공합니다. 에이전트를 실행할 때 single_step=True를 전달하기만 하면 됩니다(예: agent.run(your_task, single_step=True)).

> 다단계 에이전트에 대해 자세히 알아보려면 [LangChain 에이전트로서의 오픈소스 LLM](https://huggingface.co/blog/open-source-llms-as-agents){:target="\_blank"}에 관한 블로그 게시물을 읽어보세요.

[origin]: https://huggingface.co/docs/smolagents/conceptual_guides/intro_agents
