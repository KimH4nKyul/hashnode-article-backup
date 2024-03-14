---
title: "TypeORM 0.3.x에서 deprecated된 @EntityRepository 문제 해결하기"
datePublished: Tue Feb 27 2024 10:35:10 GMT+0000 (Coordinated Universal Time)
cuid: clt48eh5z000d09juauijd1oa
slug: typeorm-03x-deprecated-entityrepository
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1709090716411/9172ffdf-5601-4d58-a78a-877ca82412e9.webp
tags: nodejs, typescript, nestjs, typeorm

---

## Intro.

내 새로운 기술 스택으로 NestJS를 습득하기 위해 인프런에서 [따라하며 배우는 NestJS](https://www.inflearn.com/course/%EB%94%B0%EB%9D%BC%ED%95%98%EB%8A%94-%EB%84%A4%EC%8A%A4%ED%8A%B8-%EC%A0%9C%EC%9D%B4%EC%97%90%EC%8A%A4)라는 강의를 완강했다. 이 강의는 한글이고, 무료이며 NestJS 공식 문서를 어느정도 기반으로 했기 때문에 다른 건 찾아볼 것도 없이 이걸 완강했다.

---

## Problem.

그런데 강의에서 사용한 TypeORM이 낮은 버전이었나 보다. 강의에서 소개된 `@EntityRepository`를 사용하려고 하니