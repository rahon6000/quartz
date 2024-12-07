1. Gradle 데몬 죽이기

```
./gradlew --stop
```

2. 인텔리J 캐시 만료 + 재시작 (File -> invalidate cache -> invalidate and restart)

3. 다시 빌드

```
./gradlew clean build -x test
```
