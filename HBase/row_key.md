자기가 직접 만들어야한다..

Row key는 항상 정렬되어있고 region을 나누는 기준이 된다

Row key가 단조 증가라면, region을 정확이 어떤 곳에서 나누기 애매해지고, 하나의 region서버에 데이터가 집중될 수 있다(hotspot 현상)

따라서 다음의 규칙을 따르자

1.  Rowkey 사이즈를 줄이자
2.  단조 증가 row key를 피하자
3.  Region으로 나누기 쉬운 rowkey를 사용하자

![[Pasted image 20221231133608.png]]