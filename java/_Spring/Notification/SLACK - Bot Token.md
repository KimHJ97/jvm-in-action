# SLACK Bot Token 방식

## 1. 사전 작업

 - `SLACK API 앱 등록`
    - https://api.slack.com/apps
 - `OAuth & Permissions 탭`
    - Features > OAuth & Permissions > OAuth Tokens 정보 복사
    - Scopes > Add an OAuth Scope > 권한 추가
        - chat:write, chat:write.public, channels:join

<br/>

## 2. 전송 방식

 - `채널 ID 확인`
    - https://app.slack.com/client/워크스페이스ID/채널ID
 - `채널에 봇 접속`
    - 봇 접속: https://slack.com/api/conversations.join
    - 헤더: Authorization 헤더에 "Bearer 토큰값" 추가
    - 바디:
        - channel: 접속할 채널 ID
```bash
curl -X POST https://slack.com/api/conversations.join \
  -H "Authorization: Bearer xoxb-YOUR_BOT_TOKEN" \
  -H "Content-type: application/json; charset=utf-8" \
  --data '{"channel":"C06ABCDE123"}'
```

 - `메시지 전송`
    - 메시지 전송: https://slack.com/api/chat.postMessage
    - 헤더: Authorization 헤더에 "Bearer 토큰값" 추가
    - 바디
        - channel: 메시지를 보낼 채널 ID
        - text: 전송할 메시지 내용
        - blocks: Block Kit 형식의 메시지
        - attachements: 구형 메시지 포맷
        - thread_ts: 특정 스레드에 답장할 때 사용
        - as_user: 봇이 아닌 사용자로 보낼 때 사용
```bash
curl -X POST https://slack.com/api/chat.postMessage \
  -H "Authorization: Bearer xoxb-YOUR_BOT_TOKEN" \
  -H "Content-type: application/json; charset=utf-8" \
  --data '{"channel":"C06ABCDE123", "text": "메시지"}'
```
