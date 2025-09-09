# 멀티캠퍼스 "건강기능식품 추천 AI 챗봇"

# 1. 프로젝트 개요
"헬스 메이트"는 AI 기반의 챗봇을 누구나 사용할 수 있도록 하는 챗봇 구현을 목표로 진행한 프로젝트입니다.

### 문제 인식 및 목표
현대 사람들은 "건강"이라는 키워드에 관심을 가지고 있습니다.
이에 따라 운동, 식단 관리 이외에도 각종 영양제와 같은 건강식품의 수요도 자연스럽게 상승할 것이라고 생각했습니다.
하지만 건강기능식품을 처음 접하는 사람들은 자신에게 맞는 제품인지 확인하지 못하고 구매를 하는 경우가 많습니다.

그래서 건강기능식품 구매하고자 하는 사람들에게 실제 리뷰를 바탕으로 자신에게 딱 맞는 제품을 추천하고자 했습니다.

# 2. 시연 결과

### 메인 페이지(카테고리 선택)
<img width="550" height="750" alt="스크린샷 2025-09-09 131555" src="https://github.com/user-attachments/assets/42530557-3589-45ea-961a-fd6e28e4f719" />

### 키워드 선택
<img width="550" height="400" alt="스크린샷 2025-09-09 131615" src="https://github.com/user-attachments/assets/fb6cf309-a71b-4d12-9f43-5575b86d5293" />

### 사용자 Input
<img width="550" height="500" alt="스크린샷 2025-09-09 131630" src="https://github.com/user-attachments/assets/1396213f-0a99-4804-ae76-c2b4399e48b2" />

### 추천 결과
<img width="600" height="400" alt="스크린샷 2025-09-09 131840" src="https://github.com/user-attachments/assets/95314289-82d6-4a71-8e11-dc25c6b67a44" />
<img width="600" height="400" alt="스크린샷 2025-09-09 131853" src="https://github.com/user-attachments/assets/f09e51dd-82ff-4f7f-9e50-9ac13b6354c3" />
<img width="600" height="3400" alt="스크린샷 2025-09-09 131903" src="https://github.com/user-attachments/assets/f5f6dddc-ae48-4f0d-8289-87e76ed40b51" />

# 3. 구현 과정
프로젝트는 데이터 수집 -> 전처리 -> 감정분석 및 리뷰 요약 -> RAG 구현 -> 웹 페이지 구현 으로 진행되었습니다.
'''
# 현재 페이지
page_num = 1
page_ctl = 3

write_dt_lst = []
item_nm_lst = []
content_lst = []

# 날짜
date_cut = (datetime.now() - timedelta(days=365)).strftime('%Y%m%d')

while True:
    if page_num == 26:
        print("500개 수집 완료")
        break

    print(f'start : {page_num} page 수집 중, page_ctl:{page_ctl}')

    # 1. 셀레니움으로 html 가져오기
    html_source = driver.page_source

    # 2. bs4로 html 파싱
    soup = BeautifulSoup(html_source, 'html.parser')
    time.sleep(0.5)

    # 3. 리뷰 정보 가져오기
    reviews = soup.findAll('li', {'class': 'BnwL_cs1av'})

    # 4. 한 페이지 내에서 수집 가능한 리뷰 리스트에 저장
    for review in range(len(reviews)):
        try:
            # 4-1. 리뷰 작성일자 수집
            write_dt_raw = reviews[review].findAll('span', {'class': '_2L3vDiadT9'})[0].get_text()
            write_dt = datetime.strptime(write_dt_raw, '%y.%m.%d.').strftime('%Y%m%d')
        except Exception:
            write_dt = ''

        # 4-2. 상품명 수집
        try:
            item_nm_divs = reviews[review].findAll('div', {'class': '_2FXNMst_ak'})
            if item_nm_divs:
                item_nm_info_raw = item_nm_divs[0].get_text()

                # dl 태그 안의 텍스트 추출 (없을 수도 있음)
                dl_tag = item_nm_divs[0].find('dl', {'class': 'XbGQRlzveO'})
                item_nm_info_for_del = dl_tag.get_text() if dl_tag else ''

                # 텍스트 정제
                item_nm_info = re.sub(item_nm_info_for_del, '', item_nm_info_raw)

                # '제품 선택: ' 위치 찾기
                str_start_idx = item_nm_info.find('제품 선택: ')
                if str_start_idx != -1:
                    item_nm = item_nm_info[str_start_idx + len('제품 선택: '):].strip()
                else:
                    item_nm = item_nm_info.strip()
            else:
                item_nm = ''
        except Exception:
            item_nm = ''

        # 4-3. 리뷰내용 수집
        try:
            review_div = reviews[review].findAll('div', {'class': '_1kMfD5ErZ6'})
            if review_div:
                span_tag = review_div[0].find('span', {'class': '_2L3vDiadT9'})
                if span_tag:
                    review_content_raw = span_tag.get_text()
                    review_content = re.sub(' +', ' ', re.sub('\n', ' ', review_content_raw))
                else:
                    review_content = ''
            else:
                review_content = ''
        except Exception:
            review_content = ''

        # 4-4. 수집데이터 저장
        write_dt_lst.append(write_dt)
        item_nm_lst.append(item_nm)
        content_lst.append(review_content)

    # 5. 리뷰 수집일자 기준 데이터 확인 (최근 1년치만 수집)
    if write_dt_lst and write_dt_lst[-1] < date_cut:
        break

    # 6. 페이지 이동
    try:
        driver.find_element(By.CSS_SELECTOR, f'#REVIEW > div > div._2LvIMaBiIO > div._2g7PKvqCKe > div > div > a:nth-child({page_ctl})').click()
        time.sleep(5)
    except Exception as e:
        print(f"페이지 이동 실패: {e}")
        break

    page_num += 1
    page_ctl += 1
    if page_num % 10 == 1:
        page_ctl = 3

print('done')
'''

### 3-1. 데이터 수집

### 3-2. 전처리

### 3-3. 감정분석 및 리뷰 요약

### 3-4. RAG 구현

### 3-5. 웹 페이지 구

# 4. 향후 개선 사항

# 5. 후기
