# md\_alloc

mdadm에서 md 장치 파일을 만들 때 open 시스템 콜을 호출하고, open 시스템 콜에서 md\_probe를 호출합니다. 그럼 md\_probe가 하는 일은 뭘까요? 바로 md\_alloc을 호출하는 일입니다.

md\_alloc은 md 장치를 만드는데 핵심적인 일을 하므로 자세히 분석하겠습니다.







