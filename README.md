## 담당 서비스

### 1. 검색 페이지
검색 페이지는 검색 전과 검색 후로 나뉘며 '키워드'를 입력하여
원하는 게시글을 검색할 수 있는 기능을 제공합니다.

- 검색 전 페이지는 추천 게시글, 뉴스, 트랜드인 검색어 목록을 제공합니다.
- 검색 후 페이지는 키워드가 포함된 인기순, 최신순, 유저 목록을 제공합니다.

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

### 2. 광고 페이지
광고 페이지는 사이트를 이용하는 모든 회원들이 원하는 광고가 있을 때,
광고를 신청할 수 있는 기능을 제공합니다.

#### 2-1 등록 페이지
등록 페이지에서는 입력된 값으로 결제가 완료 된 후, DB에 저장합니다.

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

3. 전문가 페이지 (거래처)



4. 화상 채팅
"""
[chat event.js의 화상채팅 요청 event 부분]
"""
"""
[video chat ]
"""
