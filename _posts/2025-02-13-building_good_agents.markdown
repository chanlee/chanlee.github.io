---
layout: post
title: '좋은 에이전트 만들기'
date: 2025-02-13 18:05:00 +0900
categories: ai agents smolagents huggingface
published: true
---

원문 : [Building good agents][origin]{:target="\_blank"}

## 좋은 에이전트 만들기

제대로 작동하는 에이전트와 작동하지 않는 에이전트를 구축하는 것에는 엄청난 차이가 있습니다. 어떻게 하면 후자의 범주에 속하는 에이전트를 구축할 수 있을까요? 이 가이드에서는 에이전트 구축을 위한 모범 사례에 대해 설명하겠습니다.

> 에이전트 구축이 처음이라면 먼저 [에이전트 소개](https://huggingface.co/docs/smolagents/conceptual_guides/intro_agents){:target="\_blank"}와 [smolagents 가이드 투어](https://huggingface.co/docs/smolagents/guided_tour){:target="\_blank"}를 읽어보세요.

**최고의 에이전트 시스템은 가장 단순한 것입니다: 워크플로우를 최대한 단순화하세요.**

워크플로우에서 LLM에 자율성을 부여하면 오류가 발생할 위험이 있습니다.

잘 프로그래밍된 에이전트 시스템은 우수한 오류 로깅과 재시도 메커니즘을 갖추고 있어서 LLM 엔진이 자신의 실수를 스스로 수정할 수 있습니다. 하지만 LLM 오류의 위험을 최소화하려면 워크플로우를 단순화해야 합니다!

[에이전트 소개](https://huggingface.co/docs/smolagents/conceptual_guides/intro_agents){:target="\_blank"}에서 다룬 예시를 다시 살펴보겠습니다: 서핑 여행 회사의 사용자 문의에 답변하는 봇입니다. 새로운 서핑 스팟에 대한 질문을 받을 때마다 에이전트가 "여행 거리 API(travel distance API)"와 "날씨 API(weather API)"에 두 번 다른 호출을 하는 대신, "return_spot_information"이라는 하나의 통합 도구를 만들 수 있습니다. 이 함수는 두 API를 한 번에 호출하고 결합된 결과를 사용자에게 반환합니다.

이렇게 하면 비용, 지연 시간, 오류 위험이 줄어듭니다!

핵심 지침은 다음과 같습니다: LLM 호출 횟수를 최대한 줄이세요.

이를 통해 몇 가지 시사점을 얻을 수 있습니다:

-   가능하면 두 개의 도구를 하나로 그룹화해야 합니다. 위의 2건의 API 예시처럼 말입니다.
-   가능하면 로직은 에이전트의 결정이 아닌 결정론적 함수에 기반해야 합니다.

### LLM 엔진으로의 정보 흐름 개선

LLM 엔진은 방 안에 갇혀 있는 지능형 로봇과 같으며, 외부 세계와의 유일한 소통 수단은 문 아래로 전달되는 메모뿐이라는 점을 기억하세요.

프롬프트에 명시적으로 입력하지 않은 내용은 LLM이 전혀 알 수 없습니다.

따라서 먼저 작업을 매우 명확하게 정의하는 것부터 시작하세요! 에이전트는 LLM으로 구동되기 때문에, 작업 지시의 사소한 차이도 완전히 다른 결과를 가져올 수 있습니다.

그 다음, 도구 사용 시 에이전트에게 전달되는 정보의 흐름을 개선하세요.

따라야 할 구체적인 지침:

-   각 도구는 LLM 엔진에 도움이 될 수 있는 모든 정보를 로깅(log)해야 합니다(도구의 forward 메서드 내에서 간단히 print 문을 사용).
-   특히 도구 실행 중 발생하는 오류에 대한 상세한 로깅이 큰 도움이 될 것입니다!

예를 들어, 위치와 날짜-시간을 기반으로 날씨 데이터를 검색하는 도구가 있습니다:

먼저, 다음은 좋지 않은 버전입니다:

```python
import datetime
from smolagents import tool

def get_weather_report_at_coordinates(coordinates, date_time):
    # Dummy function, returns a list of [temperature in °C, risk of rain on a scale 0-1, wave height in m]
    return [28.0, 0.35, 0.85]

def convert_location_to_coordinates(location):
    # Returns dummy coordinates
    return [3.3, -42.0]

@tool
def get_weather_api(location: str, date_time: str) -> str:
    """
    Returns the weather report.

    Args:
        location: the name of the place that you want the weather for.
        date_time: the date and time for which you want the report.
    """
    lon, lat = convert_location_to_coordinates(location)
    date_time = datetime.strptime(date_time)
    return str(get_weather_report_at_coordinates((lon, lat), date_time))
```

무엇이 문제인가요?

-   date_time에 사용해야 할 형식에 대한 정확한 명세가 없습니다
-   location 지정 방법에 대한 상세 설명이 없습니다
-   location이 올바른 형식이 아니거나 date_time이 제대로 포맷되지 않은 경우와 같은 실패 사례를 명시적으로 처리하는 로깅 메커니즘이 없습니다
-   출력 형식이 이해하기 어렵습니다

도구 호출이 실패할 경우, 메모리에 기록된 오류 추적으로 LLM이 도구를 역설계하여 오류를 수정할 수 있습니다. 하지만 왜 그렇게 많은 작업을 LLM에게 맡겨두어야 할까요?

이 도구를 더 나은 방식으로 구축하려면 다음과 같이 했어야 합니다:

```python
@tool
def get_weather_api(location: str, date_time: str) -> str:
    """
    Returns the weather report.

    Args:
        location: the name of the place that you want the weather for. Should be a place name, followed by possibly a city name, then a country, like "Anchor Point, Taghazout, Morocco".
        date_time: the date and time for which you want the report, formatted as '%m/%d/%y %H:%M:%S'.
    """
    lon, lat = convert_location_to_coordinates(location)
    try:
        date_time = datetime.strptime(date_time)
    except Exception as e:
        raise ValueError("Conversion of `date_time` to datetime format failed, make sure to provide a string in format '%m/%d/%y %H:%M:%S'. Full trace:" + str(e))
    temperature_celsius, risk_of_rain, wave_height = get_weather_report_at_coordinates((lon, lat), date_time)
    return f"Weather report for {location}, {date_time}: Temperature will be {temperature_celsius}°C, risk of rain is {risk_of_rain*100:.0f}%, wave height is {wave_height}m."
```

일반적으로 LLM의 부담을 줄이기 위해서는 다음과 같은 질문을 스스로에게 해보시면 좋습니다: "만약 내가 초보자이고 이 도구를 처음 사용하는 상황이라면, 이 도구로 프로그래밍하고 내 오류를 직접 수정하는 것이 얼마나 쉬울까?"

### 에이전트에 더 많은 인자 전달하기

작업을 설명하는 단순한 문자열 외에도 에이전트에 추가 객체를 전달하려면 additional_args 인자를 사용하여 모든 유형의 객체를 전달할 수 있습니다:

```python
from smolagents import CodeAgent, HfApiModel

model_id = "meta-llama/Llama-3.3-70B-Instruct"

agent = CodeAgent(tools=[], model=HfApiModel(model_id=model_id), add_base_tools=True)

agent.run(
    "Why does Mike not know many people in New York?",
    additional_args={"mp3_sound_file_url":'https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/transformers/recording.mp3'}
)
```

예를 들어 이 additional_args 인수를 사용하여 에이전트가 활용할 이미지나 문자열을 전달할 수 있습니다.

### 에이전트 디버그 하기

#### 1. 더 강력한 LLM 사용하기

에이전트 워크플로우에서 발생하는 오류 중 일부는 실제 오류이고, 다른 일부는 LLM 엔진이 제대로 추론하지 못해 발생하는 오류입니다. 예를 들어, 자동차 그림을 생성해달라고 요청한 CodeAgent의 다음 추적을 살펴보세요:

```
==================================================================================================== New task ====================================================================================================
Make me a cool car picture
──────────────────────────────────────────────────────────────────────────────────────────────────── New step ────────────────────────────────────────────────────────────────────────────────────────────────────
Agent is executing the code below: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
image_generator(prompt="A cool, futuristic sports car with LED headlights, aerodynamic design, and vibrant color, high-res, photorealistic")
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Last output from code snippet: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png
Step 1:

- Time taken: 16.35 seconds
- Input tokens: 1,383
- Output tokens: 77
──────────────────────────────────────────────────────────────────────────────────────────────────── New step ────────────────────────────────────────────────────────────────────────────────────────────────────
Agent is executing the code below: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
final_answer("/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png")
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
Print outputs:

Last output from code snippet: ───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png
Final answer:
/var/folders/6m/9b1tts6d5w960j80wbw9tx3m0000gn/T/tmpx09qfsdd/652f0007-3ee9-44e2-94ac-90dae6bb89a4.png
```

사용자는 이미지가 반환되는 대신 경로가 반환되는 것을 보게 됩니다. 이는 시스템의 버그처럼 보일 수 있지만, 실제로 에이전트 시스템이 오류를 일으킨 것이 아닙니다. 단지 LLM 두뇌가 이미지 출력을 변수에 저장하지 않는 실수를 했을 뿐입니다. 따라서 이미지를 저장할 때 기록된 경로를 활용하는 것 외에는 이미지에 다시 접근할 수 없어서, 결과적으로 이미지 대신 경로를 반환하게 되는 것입니다.

그러므로 에이전트를 디버깅하는 첫 번째 단계는 "더 강력한 LLM을 사용하는 것"입니다. Qwen2/5-72B-Instruct와 같은 다른 모델들을 사용했다면 이러한 실수는 발생하지 않았을 것입니다.

[origin]: https://huggingface.co/docs/smolagents/tutorials/building_good_agents

#### 2. 더 많은 안내/정보를 제공해 주세요
