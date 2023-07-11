---
category:
  - DirectX
  - CS
  - Cpp
---

> GPGPU : General-Purpose Graphics Processing Unit

GPGPU는 GPU를 이용하여 일반 목적, 즉 본래의 용도인 3D그래픽 처리가 아닌,   
범용적이지만 복잡한 계산을 병렬 처리를 통해 GPU의 ALU에 넘겨줘서 작업을 수행하는 기술이다.   

GPGPU를 위한 API 및 프레임워크로는 CUDA, OpenCL, DirectCompute 등이 있다.   
본래 프레임워크를 구동시키는데 유리하게 가져가기 위해서 GPU가 나왔기 때문에, 프레임워크에서 사용된다고 보면 된다.   

GPGPU 작업은 작업을 작은 단위로 분할하고, 병렬로 실행하는 방식으로 코드를 작성해야 한다.   
GPU에서 실행되는 코드를 커널이라고 보면 된다.   
이와 같은 커널은 또다시 쓰레드라는 작은 작업 단위들로 구성된다.   

CPU와 GPU에서 사용하는 메모리는 당연히 다르기 때문에 메모리 최적화도 고려해야 한다.   
병렬화 시켜야 하기 때문에 적절한 알고리즘도 선택해야 한다.   

>DirectX 에서의 ComputeShader

Direct의 GPGPU 작업은 DirectX SDK에서 제공되며, C++ 또는 HLSL (High-Level Shader Language)을 주로 사용한다.   
당연히 윈도우 환경에서 최적화 되어있다는 특징이 있다.   

아래는 GPGPU의 예시 코드이다.   

```c++
int main()
{
    // DirectX 11 디바이스 및 컨텍스트 생성
    ID3D11Device* device = nullptr;
    ID3D11DeviceContext* context = nullptr;
    D3D11CreateDevice(nullptr, D3D_DRIVER_TYPE_HARDWARE, nullptr, 0, nullptr, 0, D3D11_SDK_VERSION, &device, nullptr, &context);

    // Compute Shader 로드
    ID3D11ComputeShader* computeShader = nullptr;
    D3DReadFileToBlob(shaderFilePath, &shaderBlob); // 쉐이더 파일을 바이트 코드로 읽어옴
    device->CreateComputeShader(shaderBlob->GetBufferPointer(), shaderBlob->GetBufferSize(), nullptr, &computeShader);

    // Compute Shader 실행
    context->CSSetShader(computeShader, nullptr, 0);
    context->Dispatch(1, 1, 1); // 쓰레드 그룹의 크기 지정 (여기서는 1x1x1)

    // GPU 작업 완료 대기
    context->Flush();
    context->Finish();

    // 자원 해제
    computeShader->Release();
    context->Release();
    device->Release();

    return 0;
}
```

이에 따른 메모리를 넘겨주는 코드도 작성해야 한다.   
```c++
// 입력 데이터 준비
float inputData[] = { 1.0f, 2.0f, 3.0f };
int dataSize = sizeof(inputData);

// GPU 버퍼 생성
D3D11_BUFFER_DESC bufferDesc;
bufferDesc.ByteWidth = dataSize;
bufferDesc.Usage = D3D11_USAGE_DEFAULT;
bufferDesc.BindFlags = D3D11_BIND_SHADER_RESOURCE;
bufferDesc.CPUAccessFlags = 0;
bufferDesc.MiscFlags = 0;

D3D11_SUBRESOURCE_DATA initData;
initData.pSysMem = inputData;
initData.SysMemPitch = 0;
initData.SysMemSlicePitch = 0;

ID3D11Buffer* inputBuffer = nullptr;
device->CreateBuffer(&bufferDesc, &initData, &inputBuffer);

// Compute Shader에서 사용할 수 있도록 입력 버퍼를 설정
context->CSSetShaderResources(0, 1, &inputBuffer);
```

이제 GPU 에서 출력된 데이터를 CPU 버퍼로 복사하는 과정도 필요하다.   

```c++
// 출력 데이터를 저장할 GPU 버퍼 생성
D3D11_BUFFER_DESC outputBufferDesc;
outputBufferDesc.ByteWidth = dataSize;
outputBufferDesc.Usage = D3D11_USAGE_STAGING;
outputBufferDesc.BindFlags = 0;
outputBufferDesc.CPUAccessFlags = D3D11_CPU_ACCESS_READ;
outputBufferDesc.MiscFlags = 0;

ID3D11Buffer* outputBuffer = nullptr;
device->CreateBuffer(&outputBufferDesc, nullptr, &outputBuffer);

// Compute Shader에서 사용할 수 있도록 출력 버퍼를 설정
context->CSSetUnorderedAccessViews(0, 1, &outputBuffer, nullptr);

// Compute Shader 실행

// 출력 데이터를 CPU로 복사
float outputData[] = {};
D3D11_MAPPED_SUBRESOURCE mappedResource;
context->Map(outputBuffer, 0, D3D11_MAP_READ, 0, &mappedResource);
memcpy(outputData, mappedResource.pData, dataSize);
context->Unmap(outputBuffer, 0);

// 결과 데이터 사용
// ...
```

>마치며

프레임워크에 따라 필요한 연산이 다를것이며, 게임에서도 드로우콜을 제외하고도   
단순 반복을 통한 계산이 필요한 알고리즘의 경우에는, 이와같은 로직을 고려해볼 수 있기에   
평소에 메모리와 오버헤드를 정확하게 인지하고 있는것이 중요한 부분이 되겠다.   