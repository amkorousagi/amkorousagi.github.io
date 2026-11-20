---
layout: post
tags: ["Flutter", "Dart", "카메라", "이미지 처리", "iOS", "온디바이스 추론"]
title: "Flutter 카메라 프레임을 이미지로 바꿨더니 옆으로 밀렸다 — Uint8List.buffer와 행 패딩"
description: iOS BGRA8888 카메라 프레임을 image 패키지로 바꿀 때 화면이 비스듬히 밀리는 원인. 뷰의 오프셋을 무시하는 .buffer, 그리고 bytesPerRow 패딩
---
Flutter `camera` 플러그인으로 받은 iOS 프레임(`CameraImage`, BGRA8888)을 `image` 패키지의 `Image`로 바꿔 모델 입력에 쓰려고 했습니다. 결과 이미지가 **옆으로 밀리고 비스듬하게 찢어져** 나왔습니다. 원인은 두 가지가 겹친 것이었습니다.

## 문제가 된 코드

```dart
img.Image convert(CameraImage image) {
  return img.Image.fromBytes(
    width: image.width,
    height: image.height,
    bytes: image.planes[0].bytes.buffer,   // ← 여기
    order: img.ChannelOrder.bgra,
  );
}
```

## 원인 1: `.buffer`는 뷰가 아니라 버퍼 전체다

Dart의 `Uint8List`는 `ByteBuffer` 위에 얹힌 **뷰**입니다. 뷰는 버퍼의 어느 지점부터 시작하는지(`offsetInBytes`)와 길이(`lengthInBytes`)를 갖고 있습니다.

`.buffer`를 넘기면 이 정보가 사라지고 **버퍼의 처음부터** 읽습니다. 플러그인이 픽셀 앞에 다른 데이터를 둔 버퍼를 뷰로 건네주면, 그 앞부분이 첫 픽셀인 것처럼 읽혀 이미지 전체가 몇 픽셀씩 밀립니다. 앞에 헤더가 붙은 것처럼 보이는 이유입니다.

뷰를 그대로 쓰거나, `.buffer`가 필요한 API라면 오프셋을 같이 넘깁니다.

```dart
final plane = image.planes[0];
final bytes = plane.bytes;                         // 뷰 그대로
// 또는: bytes.buffer.asUint8List(bytes.offsetInBytes, bytes.lengthInBytes)
```

## 원인 2: 한 행의 바이트 수는 width × 4가 아닐 수 있다

카메라 버퍼는 메모리 정렬 때문에 행 끝에 패딩을 붙이는 경우가 많습니다. 그래서 한 행의 실제 길이는 `plane.bytesPerRow`이고, `width * 4`보다 클 수 있습니다. 패딩을 무시하고 `width * 4`씩 끊어 읽으면 행마다 조금씩 어긋나 이미지가 **대각선으로 기울어집니다**.

## 고친 코드

```dart
img.Image convertBgra(CameraImage cam) {
  final w = cam.width, h = cam.height;
  final plane = cam.planes[0];
  final bytes = plane.bytes;              // 뷰 그대로 사용
  final stride = plane.bytesPerRow;       // 패딩 포함 행 길이
  final out = img.Image(width: w, height: h);

  for (var y = 0; y < h; y++) {
    final row = y * stride;
    for (var x = 0; x < w; x++) {
      final i = row + x * 4;
      out.setPixelRgb(x, y, bytes[i + 2], bytes[i + 1], bytes[i]); // B,G,R → R,G,B
    }
  }
  return out;
}
```

## 실시간 추론이라면

위 코드는 원인을 보여 주기 위한 것이고, Dart에서 픽셀마다 루프를 돌면 프레임마다 수십 ms가 걸립니다. 실시간으로 쓰려면 다음을 고려합니다.

- `Image` 객체를 거치지 않고 **바로 모델 입력 텐서(Float32List)**를 채운다
- 크롭·리사이즈를 먼저 해서 처리할 픽셀 수를 줄인다
- 무거운 변환은 네이티브(플랫폼 채널, FFI)나 추론 그래프 안으로 옮긴다
- 매 프레임 버퍼를 새로 만들지 않고 재사용한다. 텐서 재할당만 없애도 프레임당 시간이 눈에 띄게 줄어듭니다([onnxruntime_flutter 수정](https://github.com/amkorousagi/onnxruntime_flutter))

## 정리

- `Uint8List.buffer`는 뷰의 시작 위치를 버린다. 뷰를 그대로 쓰거나 오프셋을 같이 넘긴다
- 행 길이는 `bytesPerRow`를 쓴다. `width × 채널 수`로 가정하지 않는다
- 이미지가 밀리면 시작 위치를, 기울면 행 길이를 의심한다
