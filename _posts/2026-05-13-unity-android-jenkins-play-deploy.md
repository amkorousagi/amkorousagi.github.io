---
layout: post
tags: ["CI/CD", "Jenkins", "Android", "Unity", "Google Play", "LLM"]
title: "Unity Android 앱을 Jenkins로 Google Play 내부 테스트까지 자동 배포하기"
description: 키스토어·서비스 계정·Play Console 권한부터 빌드 번호 자동 증가, 첫 업로드 함정, 슬랙 알림까지 순서대로 정리
---

Unity로 만든 Android 앱을 사람이 직접 빌드해서 Play Console에 올리면, 빌드 PC 상태나 서명 설정에 따라 결과가 달라지고 QA 배포가 하루씩 밀리기도 합니다. 브랜치에 머지되면 빌드부터 Google Play 내부 테스트 트랙 업로드, 팀 알림까지 자동으로 이어지게 만든 과정을 순서대로 정리합니다.

## 전체 흐름

```text
GitHub (push / merge)
   └─ webhook ─▶ Jenkins 파이프라인
                   ├─ Unity batchmode 빌드 (BuildScript.cs) → 서명된 AAB
                   ├─ 빌드 번호(versionCode) 자동 증가
                   ├─ Google Play Developer API로 내부 테스트 트랙 업로드
                   └─ 슬랙 알림 (변경 커밋 요약 포함)
```

## 1. 서명 준비 — 키스토어

- 업로드용 키스토어와 별칭·비밀번호를 정하고, 키스토어 파일은 저장소가 아니라 **Jenkins Credentials**에 넣습니다
- Play App Signing을 쓰면 Google이 앱 서명 키를 보관하고, 우리는 업로드 키로만 서명합니다. 업로드 키를 잃어버려도 재설정할 수 있어 관리 부담이 줄어듭니다

## 2. Google Cloud 서비스 계정과 Play Console 권한

1. Google Cloud 프로젝트에서 **Google Play Android Developer API**를 켭니다
2. 서비스 계정을 만들고 JSON 키를 발급해 Jenkins Credentials에 넣습니다
3. Play Console → 사용자 및 권한에서 이 서비스 계정 이메일을 초대하고, 해당 앱에 대해 **출시 관리 권한**(테스트 트랙 업로드)을 줍니다

권한이 반영되기까지 시간이 걸릴 수 있어, 업로드가 권한 오류로 실패하면 몇 분 뒤 다시 시도해 봅니다.

## 3. 첫 버전은 반드시 손으로 올린다

API 업로드는 **Play Console에 앱의 첫 번째 번들이 수동으로 올라간 뒤부터** 동작합니다. 새 앱이라면 한 번은 콘솔에서 직접 AAB를 올리고 내부 테스트 트랙을 만들어 둡니다. 이걸 모르면 API 오류 메시지만 보고 서비스 계정 권한을 한참 의심하게 됩니다.

## 4. Unity 빌드 스크립트

에디터를 띄우지 않고 빌드하려면 `-batchmode -executeMethod`로 부를 정적 메서드를 만듭니다.

```csharp
public static class BuildScript
{
    public static void BuildAndroid()
    {
        PlayerSettings.Android.useCustomKeystore = true;
        PlayerSettings.Android.keystoreName = Env("KEYSTORE_PATH");
        PlayerSettings.Android.keystorePass = Env("KEYSTORE_PASS");
        PlayerSettings.Android.keyaliasName = Env("KEY_ALIAS");
        PlayerSettings.Android.keyaliasPass = Env("KEY_ALIAS_PASS");
        PlayerSettings.Android.bundleVersionCode = int.Parse(Env("BUILD_NUMBER"));
        EditorUserBuildSettings.buildAppBundle = true;

        var report = BuildPipeline.BuildPlayer(new BuildPlayerOptions {
            scenes = EditorBuildSettings.scenes.Where(s => s.enabled).Select(s => s.path).ToArray(),
            locationPathName = "Build/app.aab",
            target = BuildTarget.Android,
        });
        if (report.summary.result != BuildResult.Succeeded) EditorApplication.Exit(1);
    }
    static string Env(string k) => System.Environment.GetEnvironmentVariable(k);
}
```

- 비밀값은 코드에 두지 않고 Jenkins가 환경 변수로 넣어 줍니다
- 빌드 실패 시 종료 코드를 1로 내야 Jenkins 단계가 실패로 표시됩니다

## 5. 빌드 번호 자동 증가

Play는 같은 versionCode를 두 번 받지 않습니다. Jenkins의 `BUILD_NUMBER`나 별도 카운터를 versionCode로 쓰면 사람이 숫자를 올리는 일을 없앨 수 있습니다.

주의할 점은 **트랙과 브랜치가 여러 개일 때 versionCode가 Play 전체에서 하나의 수열**이라는 것입니다. 개발 브랜치에서 낮은 번호를 넣어도 Play에 이미 더 큰 번호가 있으면 거절되므로, 업로드 직전에 Play의 최신 versionCode를 조회해 +1 하는 방식이 가장 안전합니다.

## 6. Jenkinsfile

```groovy
pipeline {
  agent any
  environment { UNITY = '/path/to/Unity' }
  stages {
    stage('Build') {
      steps {
        withCredentials([file(credentialsId: 'upload-keystore', variable: 'KEYSTORE_PATH'),
                         string(credentialsId: 'keystore-pass', variable: 'KEYSTORE_PASS')]) {
          sh '"$UNITY" -batchmode -quit -nographics -projectPath . -executeMethod BuildScript.BuildAndroid -logFile -'
        }
      }
    }
    stage('Upload') {
      steps {
        withCredentials([file(credentialsId: 'play-service-account', variable: 'SA_JSON')]) {
          sh 'python3 tools/upload_internal.py --json "$SA_JSON" --aab Build/app.aab'
        }
      }
    }
  }
  post { always { slackSend message: "${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.currentResult}" } }
}
```

처음 실행하면 Groovy 샌드박스가 일부 메서드 호출을 막는데, Jenkins 관리 화면의 스크립트 승인에서 허용해 줍니다.

## 7. 알림에 "무엇이 바뀌었나" 붙이기

빌드 성공 알림만으로는 QA가 무엇을 확인해야 할지 모릅니다. GitHub REST API의 compare(`/repos/{owner}/{repo}/compare/{base}...{head}`)로 직전 배포 이후 커밋 목록을 가져와 알림에 붙였습니다. 커밋이 많을 때는 로컬에서 돌린 작은 LLM으로 몇 줄 요약을 만들어 함께 보냈습니다. 코드가 외부로 나가지 않는다는 점에서 로컬 모델이 편했습니다.

## 정리

- 서명 키·서비스 계정 키는 전부 CI 비밀 저장소에, 코드에는 환경 변수 이름만
- 첫 번들은 수동 업로드, 이후부터 API
- versionCode는 사람이 아니라 파이프라인이 정한다
- 알림에는 결과뿐 아니라 변경 내용을 붙인다
