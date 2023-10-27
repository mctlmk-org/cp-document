### ----- ###
- 이슈 1  
클러스터간 Proxy통신을 위해서는 PodSecurity Privileged 를 허용해야 한다.
최종 배포판에서는 Privileged 모드로 진행하고, gitHub에 관련 가이드  한다. 
=> ex) PodSecurity 모드를 적용할 경우 Multi Cluster 간에 사이드카 프록시 통신이 제한되기 때문에 기본 Privileged모드로 진행한다. 운영상의 환경에서는 별도의 조치를 취해야 한다.   
담당자 : 김태우

- 이슈 2   
ConfigMap(인증서, 인증서 배치를 위한 명령어)을 데몬셋을 배포하면 각 노드에 Harbor인증서를 배포할 수 있다. 다만 실행되는 Namespace는 PodSecurity가 Privileged 상태여야 한다.   
CP 배포판, KubernetesService 배포판 모두 적용한다.    
담당자 : 강지훈

- 이슈 3
vault policy
ingress-controller의 pod_IP만 허용IP 로 등록해도 모든 경로에서 허용된다. 
해당 IP를 제외하면 ?? --> 허용안됨
--> 요청하는 구간에서의 IP만 허용되도록 테스트 필요하다. 
--> root Token 폐기하고,  application 별 Token을 사용하기 때문에 괜찮은지 확인 필요.
담당자 : 강지훈

- 이슈 4
Istio  
멀티 클라우드 간 설치 시 Istio 중복(KubeFlow)
멀티 클라우드 설치 시 KubeFlow 설치 해제한다. 
싱글 배포일때도 KubeFlow은 별도 배포스크립트를 뺀다. 

컨테이너보안  
PodSecurity : 멀티에서 Privileged / Single에서 그대로..
담당자 : 김태우
