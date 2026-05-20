# 담당 기능 소개

## 목차
1. [검색 페이지](#1-검색-페이지)
2. [광고 페이지](#2-광고-페이지)
3. [전문가 페이지 (거래처)](#3-전문가-페이지-거래처)
4. [화상 채팅](#4-화상-채팅)

---

## 1. 검색 페이지

키워드를 입력하여 원하는 게시글을 탐색할 수 있는 기능입니다.  
검색 전 · 검색 후 두 가지 상태로 화면이 구성됩니다.

### 검색 전

| 항목 | 설명 |
|---|---|
| 추천 게시글 | 사용자 맞춤 추천 게시글 목록 제공 |
| 뉴스 | 최신 뉴스 목록 제공 |
| 트렌드 검색어 | 현재 인기 있는 검색어 목록 제공 |

### 검색 후

| 항목 | 설명 |
|---|---|
| 인기순 | 좋아요 수 기준으로 정렬된 게시글 목록 |
| 최신순 | 최근 등록 순으로 정렬된 게시글 목록 |
| 유저 목록 | 키워드와 연관된 사용자 목록 |

### 주요 코드

**추천 게시글 조회 API** — 로그인한 사용자 기준으로 추천 게시글을 페이징하여 반환하며, S3에 저장된 이미지는 Presigned URL로 변환하여 제공합니다.

```java
@GetMapping("products/{page}")
public ResponseEntity<?> getRecommends(@PathVariable int page,
                                       @AuthenticationPrincipal CustomUserDetails userDetails) {
    PostProductWithPagingDTO posts = postProductService.getRecommendProducts(page, userDetails.getId());

    posts.getPosts().forEach(post -> {
        if (post.getMemberProfile() != null && !post.getMemberProfile().isBlank()) {
            post.setMemberProfile(
                    toPresignedUrlOrOriginal(post.getMemberProfile())
            );
        }

        if(post.getPostFiles() == null || post.getPostFiles().isEmpty()) {
            return;
        }
        post.setPostFiles(
                post.getPostFiles().stream()
                        .map(this::toPresignedUrlOrOriginal)
                        .collect(Collectors.toList())
        );

    });

    return ResponseEntity.ok(posts);
}
```

---

## 2. 광고 페이지

사이트 회원이라면 누구든지 광고를 신청·조회할 수 있는 기능입니다.  
등록, 목록 조회, 상세 조회 세 가지 화면으로 구성됩니다.

### 광고 등록

결제 완료 후 광고 정보와 이미지를 DB 및 S3에 저장합니다.

| 단계 | 설명 |
|---|---|
| 1. 정보 입력 | 광고 제목, 내용, 이미지 등 정보 입력 |
| 2. 결제 처리 | 결제 완료 후 등록 진행 |
| 3. 데이터 저장 | 광고 정보는 DB, 이미지는 S3에 저장 |

> **예외 처리** — 이미지 업로드 중 오류 발생 시, 업로드된 파일과 광고 데이터를 모두 롤백하여 데이터 정합성을 보장합니다.

```java
@PostMapping("write")
@LogStatusWithReturn
public ResponseEntity<?> write(@ModelAttribute AdvertisementDTO advertisementDTO,
                               @RequestParam(value = "images", required = false) ArrayList<MultipartFile> images,
                               @AuthenticationPrincipal CustomUserDetails userDetails) throws IOException {
    // 유저 id 받아와서 주입
    advertisementDTO.setAdvertiserId(userDetails.getId());
    advertisementService.save(advertisementDTO);

    if (images != null && !images.isEmpty()) {
        String todayPath = advertisementService.getTodayPath();
        List<String> uploadedKeys = new ArrayList<>();

        try {
            for (MultipartFile image : images) {
                String s3Key = s3Service.uploadFile(image, todayPath);
                uploadedKeys.add(s3Key);
                advertisementService.saveFile(advertisementDTO.getId(), image, s3Key);
            }
        } catch (Exception e) {
            uploadedKeys.forEach(s3Service::deleteFile);
            advertisementService.delete(advertisementDTO.getId());
            throw new RuntimeException("파일 업로드 실패", e);
        }
    }

    log.info("저장된 광고 ID: {}", advertisementDTO.getId());

    Map<String, Object> result = new HashMap<>();
    result.put("id", advertisementDTO.getId());
    result.put("message", "광고 등록 성공!");

    return ResponseEntity.ok(result);
}
```

### 광고 조회

등록된 광고 목록을 확인할 수 있습니다.

```java
@GetMapping("list/{page}")
public ResponseEntity<?> list(@PathVariable int page, AdSearch search,
                              @AuthenticationPrincipal CustomUserDetails userDetails) {

    AdWithPagingDTO result = advertisementService.list(page, search, userDetails.getId());

    result.getAdvertisements().forEach(ad ->
            ad.setImgUrls(convertToPresignedUrl(ad.getImgUrls()))
    );

    return ResponseEntity.ok(result);
}
```

### 광고 상세

특정 광고를 선택하면 상세 정보를 확인할 수 있습니다.

```java
@GetMapping("detail")
@LogStatusWithReturn
public ResponseEntity<?> detail(Long id) {
    AdvertisementDTO adDTO = advertisementService.getAdvertisementDetail(id);
    adDTO.setImgUrls(convertToPresignedUrl(adDTO.getImgUrls()));
    return ResponseEntity.ok(adDTO);
}
```

---

## 3. 전문가 페이지 (거래처)

사이트에 등록된 전문가 및 거래처 정보를 조회할 수 있는 기능입니다.

```java
@PostMapping("member-list/{page}")
public ResponseEntity<?> getInquiryMembers(@PathVariable int page,
                                           @RequestBody Map<String, String> body,
                                           @AuthenticationPrincipal CustomUserDetails userDetails) {
    if (userDetails == null) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(Map.of("error", "로그인이 필요합니다."));
    }
    if (userDetails.getMemberRole() != MemberRole.EXPERT) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(Map.of("error", "전문가 권한이 필요합니다."));
    }

    String categoryName = body.get("categoryName");

    InquiryMemberWithPagingDTO inquiryDTO = memberService.getInquiryMembers(page, categoryName, userDetails.getId());
    inquiryDTO.getMembers().forEach(inquiryMemberDTO -> {
        if(inquiryMemberDTO.getFilePath() != null && !inquiryMemberDTO.getFilePath().isEmpty()) {
            try {
                inquiryMemberDTO.setFilePath(s3Service.getPresignedUrl(inquiryMemberDTO.getFilePath(), Duration.ofMinutes(10)));
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    });

    return ResponseEntity.ok(inquiryDTO);
}
```

---

## 4. 화상 채팅

WebRTC 기반으로 사용자 간 실시간 화상 통화를 지원하는 기능입니다.

| 기능 | 설명 |
|---|---|
| 화상 채팅 요청 | 상대방에게 화상 채팅 요청 이벤트 전송 |
| 수락 / 거절 | 요청 수신자가 수락 또는 거절 처리 |
| 채팅 종료 | 양측 어디서든 세션 종료 가능 |

**화상 채팅 요청 이벤트**

```javascript
async function startVideoCall() {
        if (!currentMemberId || !currentPartnerId) {
            console.error("통화 상대방 정보가 없습니다.");
            return;
        }

        try {
            const response = await fetch("/api/video-chat/session", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({
                    receiverId: currentPartnerId,
                    receiverName: currentMemberName
                })
            });

            if (!response.ok) throw new Error("세션 생성 실패: " + response.status);

            const sessionData = await response.json();
            const roomName = sessionData.roomName || "conversation-" + sessionData.conversationId;
            sessionId = sessionData.id;
            console.log(sessionId);

            // [임시 - 로컬 테스트용 프록시] 배포 시 아래 requestLiveKitToken 원본으로 교체
            const token = await requestLiveKitToken(roomName);
            window.location.href = `/video-chat/join?token=${encodeURIComponent(token)}&roomName=${encodeURIComponent(roomName)}&sessionId=${sessionId}&userName=${encodeURIComponent(currentMemberName)}&partnerName=${encodeURIComponent(currentPartnerName)}&partnerId=${currentPartnerId}`;

        } catch (error) {
            console.error("화상통화 시작 실패:", error.message);
        }
    }
```

**화상 채팅 연결 처리**

```javascript
@PostMapping(value = "/token")
    public ResponseEntity<Map<String, String>> createToken(@RequestBody Map<String, String> params) {
        String roomName = params.get("roomName");
        String participantName = params.get("participantName");
        log.info("{}, {}", roomName, participantName);

        if (roomName == null || participantName == null) {
            return ResponseEntity.badRequest().body(Map.of("errorMessage", "roomName and participantName are required"));
        }

        AccessToken token = new AccessToken(LIVEKIT_API_KEY, LIVEKIT_API_SECRET);
        token.setName(participantName);
        token.setIdentity(participantName);
        token.addGrants(new RoomJoin(true), new RoomName(roomName));

        return ResponseEntity.ok(Map.of("token", token.toJwt()));
    }
```
