---
{"dg-publish":true,"permalink":"/logs/digital-garden-logs/analytics/","updated":"2024-08-23T21:46:00"}
---



vercel analytics를 붙였습니다.

```html
//_includes/user/common/footer/anaylytics.njk
<script defer src="/_vercel/insights/script.js"></script>
<script defer src="/_vercel/speed-insights/script.js"></script>
```

이 dg를 vercel로 배포하고있기 때문에 vercel의 analytics를 활용할 수 있어 완전 럭키비키자냐ㅋㅋ

해당 스크립트를 `_includes/user/common/footer/anaylytics.njk` 가상의 slot 경로에다 넣고 배포를 하면 아래와 같이 vercel 대시보드에서 방문자수를 볼 수 있습니다.

hobby용으로는 무료요금으로 확인할 수 있는 듯 하네요.

![Screen Shot 2024-08-23 at 12.37.13 AM.png|50%](/img/user/Screen%20Shot%202024-08-23%20at%2012.37.13%20AM.png)

역시 첫 방문자는 나 :0

꽤나 다양한 정보를 제공해주는군요.

![Screen Shot 2024-08-23 at 12.53.13 AM.png|50%](/img/user/Screen%20Shot%202024-08-23%20at%2012.53.13%20AM.png)

아니 어떻게 알고 들어오는거지? 다양한 나라에서.. 들어오는 중… 신기해!

![IMG_3129.png|300](/img/user/IMG_3129.png)