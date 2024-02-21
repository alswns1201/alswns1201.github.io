
## GitHub io 만들기.
> 포트폴리오 용

### 유의 사항.
- Repository name을 name.github.io 로 만들어 준다.
- jekyll를 통한 theme를 적용 할 수 있다.
  - jekyll를 쓰기 위해서는 Ruby를 설치
  - 받은 파일을 repository에 복사하여 사용.
- _config.yml을 적절하게 변경.
- Gemfile에서 gem "webrick", "~> 1.8" 을 추가해 줬다.

### _config.yml
> Code block will look like this.
```yml
highlighter-theme: monokai //you can change your syntax color scheme.
date_format: "%Y-%M-%D" //and date format.
```

### Screenshots
#### Page
![alt text](/public/img/screenshot-1.png)
#### Articles
![alt text](/public/img/screenshot-2.png)
#### Page - Mobile
![alt text](/public/img/screenshot-m1.png)
#### Page - Articles
![alt text](/public/img/screenshot-m2.png)
