# SLACK Webhook 방식

## 1. 사전 작업

 - `SLACK API 앱 등록`
    - https://api.slack.com/apps
 - `Webhooks 탭`
    - Features > Incoming Webhooks > 활성화
    - Add New Webhook 클릭

<br/>

## 2. 전송 방식

해당 Webhook URL에 POST 방식으로 JSON 값 입력하면 메시지가 전송된다.

```bash
curl -X POST https://hooks.slack.com/services/주소 \
  -H 'Content-type: application/json' \
  --data '{"text": "Webhook 방식으로 전송된 메시지입니다 💬"}'
```

<br/>

## 3. 전송 예시

 - `전송 DTO`
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SlackMessage {
    private String text;
    private String channel;
    private String username;
    
    @JsonProperty("icon_emoji")
    private String iconEmoji;
}
```

 - `SlackService`
```java
@Slf4j
@Service
@RequiredArgsConstructor
public class SlackService {
    
    private final RestTemplate restTemplate;
    
    @Value("${slack.webhook-url}")
    private String webhookUrl;
    
    public void sendMessage(String message) {
        try {
            SlackMessage slackMessage = SlackMessage.builder()
                    .text(message)
                    .build();
            
            sendMessage(slackMessage);
        } catch (Exception e) {
            log.error("Slack 메시지 전송 실패: {}", e.getMessage(), e);
        }
    }
    
    public void sendMessage(SlackMessage slackMessage) {
        try {
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            
            HttpEntity<SlackMessage> entity = new HttpEntity<>(slackMessage, headers);
            
            String response = restTemplate.postForObject(webhookUrl, entity, String.class);
            log.info("Slack 메시지 전송 성공: {}", response);
        } catch (Exception e) {
            log.error("Slack 메시지 전송 실패: {}", e.getMessage(), e);
            throw new RuntimeException("Slack 메시지 전송 중 오류 발생", e);
        }
    }
    
    // 고급 메시지 전송 (채널, 사용자명, 아이콘 지정)
    public void sendCustomMessage(String message, String channel, String username, String emoji) {
        SlackMessage slackMessage = SlackMessage.builder()
                .text(message)
                .channel(channel)
                .username(username)
                .iconEmoji(emoji)
                .build();
        
        sendMessage(slackMessage);
    }
}
```




