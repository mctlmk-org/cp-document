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

