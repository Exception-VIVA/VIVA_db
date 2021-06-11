# VIVA_db
## 목적

: 앱 내 "개인맞춤형 모의고사 추천 서비스" 및 "채점 결과 연동 오답노트 생성"서비스를 위해서는 문제와 그에 해당하는 답안지 이미지를 DB에 보유하고 있어야함.

실제 제공되는 문제집이나 문제지에서는 문제별로 이미지를 나누어 제공하지 않기 때문에 이를 생성해내야한다.

![image](https://user-images.githubusercontent.com/52443701/121131499-dd4f9180-c86a-11eb-8dce-274c21ffad28.png)

문제 및 답안 이미지 사용 예시(오답노트)

## 적용데이터 목록

저작권에 문제가 없는 EBS, 모의고사로 진행함

### 문제지/답안지

- [ ]  수능완성 (가/나) 2021 - 2017년
- [ ]  평가원 모의고사 (가/나) 2021 - 2017년 (6/9/수능)
- [ ]  교육청 모의고사 (가/나) 2021 - 2019년 (3/4/7/10월)

## 문제지 영역 자르기

![image](https://user-images.githubusercontent.com/52443701/121131665-12f47a80-c86b-11eb-88ef-be0c91e1ea0b.png)


```python

def cropTop(image):
    src=cv2.imread(image)
  
    gray = cv2.cvtColor(src, cv2.COLOR_BGR2GRAY)
    canny = cv2.Canny(gray, 5000, 1500, apertureSize = 5, L2gradient = True)
 
    lines = cv2.HoughLines(canny, 0.8, np.pi / 180, 220, srn = 100, stn = 200, min_theta = 89, max_theta = 91)

    miny=700

    for i in lines:
        rho, theta = i[0][0], i[0][1]
        a, b = np.cos(theta), np.sin(theta)
        x0, y0 = a*rho, b*rho
        
        scale = src.shape[0] + src.shape[1]

        x1 = int(x0 + scale * -b)
        y1 = int(y0 + scale * a)
        x2 = int(x0 - scale * -b)
        y2 = int(y0 - scale * a)
        #print("y1",y1,"y2",y2)
        if (y2 < miny):
            miny=y2
        cv2.line(dst, (x1, y1), (x2, y2), (0, 0, 255), 2)
        header = src[:miny, :]
        body=src[miny+10:,:]
 
        
    return header,body
```

```python
def cropCenter(img):
    src=img
    dst=src.copy
    w=int (img.shape[1]/2)
    left=src[:,:w-5]
    right=src[:,w+5:]
    contour(left)
    contour(right)
```

### contours()

![image](https://user-images.githubusercontent.com/52443701/121131715-26074a80-c86b-11eb-9f28-5173452bfd34.png)

```python
def contour(page_rl):
    print("contour")

    #이미지 흑백화 
    imgray = cv2.cvtColor(page_rl, cv2.COLOR_BGR2GRAY) 
    img2=imgray.copy()
    #이미지 이진화 (스캔본 처럼)
    blur = cv2.GaussianBlur(imgray, (3,3), 0)
    thresh = cv2.threshold(blur, 70, 255, cv2.THRESH_BINARY+cv2.THRESH_OTSU)[1]

    # Morph operations
    edge = cv2.Canny(imgray, 100, 200)
    kernel = cv2.getStructuringElement(cv2.MORPH_RECT,(1000,200))
    closed = cv2.morphologyEx(edge, cv2.MORPH_CLOSE, kernel)

    contours, hierarchy = cv2.findContours(closed.copy(),cv2.RETR_TREE, cv2.CHAIN_APPROX_SIMPLE)
    

    contours_xy = np.array(contours , dtype=object)
    contours_xy.shape
    #한페이지 내에서 문제 순서대로 불러오기
    contours=reversed(contours)

		#한페이지 내의 모든 폐곡선 범위에 대해 실행 
    top=[]
    for c in contours:

			#폐곡선 바운더리 
        x,y,w,h = cv2.boundingRect(c)
        top.append(y)
  
    total=len(top)-1
    global qnum
    for i in range(total):
        qnum+=1
        if (i==0):
            img_trim=page_rl[top[i]:top[i+1]-5,:]
        else:
            img_trim=page_rl[top[i]-10:top[i+1]-5,:]
        print(qnum)

```

## 답안지 데이터 생성

### Input

![image](https://user-images.githubusercontent.com/52443701/121131773-36b7c080-c86b-11eb-9158-7bd3db3a6361.png)
입력 이미지의 예시 

### 탬플릿 매칭

답지 이미지 내에서 답을 표기하는 맨 하단을 기준으로 이미지를 자른다.

![image](https://user-images.githubusercontent.com/52443701/121131815-46cfa000-c86b-11eb-9c13-c86f0f4aaab5.png)


해당 이미지를 이용하여, 기준을 잡고 나눈다

```python
# 탬플릿 매칭 해내기 (자를이미지, 답 이미지)
res = cv2.matchTemplate(img_rgb, template, cv2.TM_CCOEFF_NORMED)

# 임계치 정하기 
threshold = .65

#임계치 이상만 배열에 저장
loc = np.where(res >= threshold)
```

다음과 같이 답이미지가 존재하는 곳을 찾아내어 위치값을 받아와 그를 기준으로 잘라낸다

### template matching시각화

![image](https://user-images.githubusercontent.com/52443701/121131859-564ee900-c86b-11eb-9b2a-68e1576b6eeb.png)


### flag를 이용하여 이미지 합치기

한 문제에 대한 답이 여러 페이지에 걸쳐 있을 경우 이를 합쳐주어야 함

위에서 template matching방법을 기준으로 자른 이미지를 스캔하며 "답"이미지가 없는 경우 플래그를 이용해 없는 것들과 있는 것을 합쳐 하나로 제작 

- 이미지 내 template이 존재
- 이미지 내 template이 존재 x

```python
#"답"이미지가 없는 이미지의 경우 
    if (length <= 0):
    #이미지의 경로를 image_arr배열에 저장
        image_arr[noans_count] = image_url
        #배열 index를 증가시킴 
        noans_count=noans_count+1
        #flag를 0으로 설정
        flag = 0
        
    #배열에 이미지 경로가 저장되어 있고(flag = 0 ) "답"이미지를 가진 이미지를 찾은 경우
    elif (length >0 and flag ==0):
        #여태까지 배열에 저장된 모든 이미지를 이어줌 
        for i in range(noans_count-1,-1,-1):
            image_noans = cv2.imread(image_arr[i])
            image = cv2.vconcat([image_noans,image])
            noans_count = noans_count-1

        #이어서 하나로 만든 이미지를 저장 
        include(image,qnum)
        #배열 index를 0으로 초기화 
        if (noans_count < 0):
            noans_count = 0
        #flag를 1으로 설정 
        flag = 1
    #이미지가 온전한 (문제번호~"답"이미지까지 존재하는) 이미지인 경우
    else:
        #이미지를 저장 
        include(image,qnum) 
        flag=1
```

### Output
![image](https://user-images.githubusercontent.com/52443701/121131910-6a92e600-c86b-11eb-8018-a94f634bf29d.png)

flag 까지 완료된 output의 예시


## 관려 기술 블로그
https://iagreebut.tistory.com/category/졸업프로젝트/OpenCV
