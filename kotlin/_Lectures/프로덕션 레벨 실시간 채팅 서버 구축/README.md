# 프로덕션 레벨 실시간 채팅 서버 구축: 분산 처리부터 성능 최적화까지 (Kotlin & Spring)

 - 강의 주소: https://www.inflearn.com/course/%ED%94%84%EB%A1%9C%EB%8D%95%EC%85%98-%EB%A0%88%EB%B2%A8-%EC%8B%A4%EC%8B%9C%EA%B0%84-%EC%B1%84%ED%8C%85-%EC%84%9C%EB%B2%84-%EA%B5%AC%EC%B6%95/dashboard

<br/>

## 학습 내용

 - Redis Pub/Sub을 활용하여 여러 서버 인스턴스 간 실시간 메시지를 동기화하는 분산 메시징 시스템
 - Docker Compose로 3개 인스턴스 클러스터를 구성하고 Nginx 로드밸런서로 트래픽을 분산 처리하는 방법
 - Spring WebSocket을 이용하여 양방향 실시간 통신을 구현
 - 멀티모듈 구조로 Domain-driven Design을 적용하여 확장 가능한 서버 아키텍처를 설계하는 방법
 - Redis를 활용한 메시지 시퀀스 관리로 분산 환경에서도 메시지 순서를 보장하는 기법
 - Docker 컨테이너 기반 배포와 헬스체크, 로그 모니터링을 통한 서비스 운영 방법
 - Spring Boot 3.x + Kotlin의 최신 기능을 활용한 모던 백엔드 개발 방법
 - 메시지 중복 처리 방지, 세션 정리, 리소스 해제 등 안정적인 서비스 운영을 위한 방어 코딩 기법
