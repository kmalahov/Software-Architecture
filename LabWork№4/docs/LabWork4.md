# Лабораторная работа №4
Проектирование REST API. <br/>
Деменева Анастасия, Малахов Константин, Шутов Артемий <br/>
Для доступа к запросам необходима аутентификация. Пожалуйста, предоставьте токен доступа в заголовке запроса (Authorization: Bearer YOUR_ACCESS_TOKEN).
# GET запросы
## 1. Данный запрос выводит на экран список всех презентаций пользователя
### Метод: 
GET
### Путь: 
/presentations
### Параметры запроса:
 - limit (необязательный, по умолчанию 10): Количество записей на странице.
 - page (необязательный, по умолчанию 1): Номер страницы.
 - sort (необязательный, по умолчанию "date_creation"): Параметр сортировки. Возможные значения: "name", "name-reverse", "date_creation", "date_creation-reverse".
### Ответ:
В случае успешного запроса, сервер возвращает JSON-объект, содержащий следующие поля:
 - total_count: Общее количество презентаций пользователя в базе данных.
 - presentations: Список презентаций, соответствующих запросу.<br/>

Каждая презентация в списке представлена объектом со следующими полями:
 - id: Уникальный идентификатор презентации.
 - name: Название презентации.
 - date_creation: Дата создания презентации.
 - visible: Флаг видимости презентации.
 - presentation_logo: Изображение-логотип презентации.
 - count_questions: Общее количество вопросов в презентации.
 - count_slides: Общее количество слайдов в презентации.

![image_2024-02-04_22-50-38](https://github.com/AnaSKBK/PAPS/assets/128895913/6872692f-0b8a-46b0-9a6e-d7de9b88a2cd)

### Пример использования:
curl -X GET <br/>
"https://backend-test.diaclass.kz/presentations?limit=10&page=1&sort=date_creation" -H "Authorization: Bearer YOUR_ACCESS_TOKEN"

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка аутентификации",<br/>
 "code": 401,<br/>
 "type": "unauthorized"<br/>
}

### Реализация API:
```
@router.get("/presentations")
async def get_presentation(
       limit: Optional[int] = 10,
       page: Optional[int] = 1,
       sort: Optional[str] = "date_creation",
       db: AsyncSession = Depends(get_db),
       current_user: users.User = Depends(user_requests.get_current_user_by_auth)):
   # Параметры сортировки
   sort_options = {
       "name": lambda p: (p["name"], -p["date_creation"].timestamp()),
       "name-reverse": lambda p: (p["name"], p["date_creation"]),  # -p.name,
       "date_creation": lambda p: -p["date_creation"].timestamp(),
       "date_creation-reverse": lambda p: -p["date_creation"].timestamp(),
   }
   # Получение начального и конечного индексов на основе страницы и лимита
   start_idx = (page - 1) * limit
   end_idx = start_idx + limit
   # Основной запрос для выборки всех презентаций и их связанных слайдов и вопросов
   main_query = (select())
   result = await db.execute(main_query)
   presentation_list = [presentation for presentation in result.scalars()]
   result_list = [
       {
           "id": p.id,
           "name": p.name,
           "date_creation": p.date_creation,
           "visible": p.visible,
           "presentation_logo": p.presentation_logo,
           "count_questions": sum(len(slide.questions) for slide in p.slides),
           "count_slides": len(p.slides)
       }
       for p in presentation_list
   ]
   sorted_presentation = sorted(result_list, key=sort_options[sort], reverse=sort.endswith("-reverse"))
   filtered_presentation = sorted_presentation[start_idx:end_idx]


   # Общее количество записей в БД
   total_count = await db.scalar(select(func.count()).select_from(main_query.subquery()))
   await db.close()
   return {"total_count": total_count, "presentations": filtered_presentation}
```
## 2. Данный запрос позволяет получить вопрос для конкретного слайда презентации
### Метод: 
GET
### Путь: 
/presentation/slide/question
### Параметры запроса:
 - slide_id: Идентификатор слайда, для которого нужно получить вопрос.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Аутентифицированный пользователь (автоматически предоставляется зависимостью).
### Ответ:
В случае успешного выполнения запроса, сервер возвращает объект вопроса для указанного слайда вместе с идентификатором типа слайда. <br/>
Если для указанного слайда не существует вопроса, возвращается объект с идентификатором типа слайда.
![image_2024-02-04_22-52-14](https://github.com/AnaSKBK/PAPS/assets/128895913/b718aeb6-12a7-4d90-8b07-9e79b852a0c5)


### Пример использования:
curl -X GET <br/>
"https://backend-test.diaclass.kz/presentation/slide/question?slide_id=4206"

### Реализация API:
```
@router.get("/presentation/slide/question", status_code=200)
async def get_slide_questions(
       slide_id: int,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, slide_id=slide_id)
   res = await presentation_requests.get_slide_question(db, slide_id)
   return res
```
## 3. Данный запрос позволяет получить статистику для конкретной сессии.
### Метод: 
GET
### Путь: 
/get/session/stat
### Параметры запроса:
 - session_id: Идентификатор сессии, для которой нужно получить статистику.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Аутентифицированный пользователь (автоматически предоставляется зависимостью).

### Ответ:
В случае успешного выполнения запроса, сервер возвращает словарь со статистикой для указанной сессии.<br/> 
Структура словаря:<br/> 
 - date_start: Дата и время начала сессии.
 - presentation_name: Название презентации, связанной с сессией.
 - session_code: Код сессии.
 - count_participants: Количество участников сессии.
 - count_questions: Общее количество вопросов в презентации.
 - pie_results: Процентное соотношение результатов участников (правильные, неправильные, без ответа).
 - correct: Количество правильных ответов.
 - incorrect: Количество неправильных ответов.
 - no_answer: Количество вопросов без ответа.
 - question_results: Процент правильных ответов на каждый вопрос.
 - average_score: Средний балл участников сессии.
 - participants: Список участников сессии с детальной статистикой (имя, балл, количество правильных ответов, процент правильных ответов).
 - surveys: Статистика по анкетам в презентации.
![image_2024-02-04_22-52-21](https://github.com/AnaSKBK/PAPS/assets/128895913/0011a075-ea9d-4e77-b57f-5f2b25760933)


### Пример использования:
curl -X GET <br/>
"https://backend-test.diaclass.kz/get/session/stat?session_id=2605"


### Реализация API:
```
@router.get("/get/session/stat", status_code=200)
async def get_session_stat(
       session_id: int,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await check_admin(db, current_user, session_id=session_id)
   res = await session_requests.get_session_stat(db, session_id)
   return res
```

# POST запросы
## 4. Данный запрос создает новую презентацию
### Метод: 
POST
### Путь: 
/presentation/create
### Параметры запроса:
 - presentation: Объект, содержащий информацию о создаваемой презентации.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Текущий аутентифицированный пользователь (автоматически предоставляется зависимостью).

### Ответ:
В случае успешного запроса, сервер возвращает JSON-объект с информацией о созданной презентации.
![image_2024-02-04_22-51-16](https://github.com/AnaSKBK/PAPS/assets/128895913/ef1a991d-7b41-4570-b591-7694cf22d26c)


### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/presentation/create" -H "Authorization: Bearer YOUR_ACCESS_TOKEN" -d '{"name": "Новая презентация", "id_category": 1}'

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка валидации",<br/>
 "code": 422,<br/>
 "type": "validation_error"<br/>
}


### Реализация API:
```
@router.post("/presentation/create", response_model=presentation_schemas.PresentationCreateReturn, status_code=201)
async def create_presentation(
       presentation: presentation_schemas.PresentationCreate,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.get_count_presentations(db, current_user)
   presentation = await presentation_requests.create_presentation(db, presentation, current_user)
   return presentation
```

## 5. Данный запрос создает новый слайд презентации
### Метод: 
POST
### Путь: 
/presentation/slide/add
### Параметры запроса:
 - slide: Объект, содержащий информацию о создаваемом слайде.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Текущий аутентифицированный пользователь (автоматически предоставляется зависимостью).


### Ответ:
В случае успешного запроса, сервер возвращает JSON-объект с информацией о созданном слайде и его содержимом.
![image_2024-02-04_22-51-41](https://github.com/AnaSKBK/PAPS/assets/128895913/f21f2fc8-8a6b-47d4-9319-46fb67a6a5b8)



### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/presentation/slide/add" -H "Authorization: Bearer YOUR_ACCESS_TOKEN" -d '{"id_presentation": 1, "id": null}'

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка проверки администратора",<br/>
 "code": 403,<br/>
 "type": "forbidden"<br/>
}


### Реализация API:
```
@router.post("/presentation/slide/add", response_model=slide_schemas.SlideWithContent, status_code=201)
async def add_slide(
       slide: slide_schemas.SlideCreate,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, presentation_id=slide.id_presentation)
   await presentation_requests.get_count_slides(db, current_user, slide.id_presentation)
   slide = await presentation_requests.add_slide(db, slide.id_presentation, slide.id)
   return slide

```
## 6. Данный запрос позволяет зарегистрировать нового пользователя
### Метод: 
POST
### Путь: 
/sign-up
### Параметры запроса:
 - user: Объект, содержащий информацию о новом пользователе.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).


### Ответ:
В случае успешного запроса возвращается строка '0k'. Письмо с подтверждением регистрации отправляется на указанный email.

![image_2024-02-04_22-51-53](https://github.com/AnaSKBK/PAPS/assets/128895913/0ae468c2-e90e-49ca-9570-85fb965ae3ea)




### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/sign-up" -d '{"email": "user@example.com", "password": "password123"}'

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка валидации",<br/>
 "code": 422,<br/>
 "type": "validation_error"<br/>
}


### Реализация API:
```
@router.delete("/presentation/slide/delete", status_code=200)
async def del_slide(
       slide: slide_schemas.SlideDelete,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, slide_id=slide.id)
   if slide.id_slide_type == 8:
       await del_image_from_slide(db, slide.id)
   slide = await presentation_requests.del_slide(db, slide.id)
   return slide
```
## 7. Данный запрос позволяет установить пароль от аккаунта после регистрации пользователя
### Метод: 
POST
### Путь: 
/set-password
### Параметры запроса:
 - activate_token: Токен активации, полученный после регистрации.
 - user: Объект, содержащий информацию о пользователе.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - x_fingerprint: Информация о браузере пользователя (автоматически предоставляется зависимостью).


### Ответ:
В случае успешной установки пароля сервер возвращает JSON-объект с токенами доступа и информацией о пользователе.

![image_2024-02-04_22-52-00](https://github.com/AnaSKBK/PAPS/assets/128895913/72138e76-8917-4320-82ff-11d4e1824001)


### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/set-password" -d '{"activate_token": "your_activation_token", "password": "new_password"}'


### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка валидации",<br/>
 "code": 422,<br/>
 "type": "validation_error"<br/>
}


### Реализация API:
```
@router.post("/set-password")
async def set_password(response: Response, activate_token: str, user: users.UserIn, db: AsyncSession = Depends(get_db),
                      x_fingerprint=Depends(get_fingerprint)):
return {'user': {**jsonable_encoder(users.UserAuth(**jsonable_encoder(db_user)))}, 'access_token': access_token,
       "expires_in": expires_in}
```
## 8. Данный запрос позволяет пользователю зарегистрироваться с помощью аккаунта Яндекс
### Метод: 
POST
### Путь: 
/yandex_user_info
### Параметры запроса:
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - x_fingerprint: Информация о браузере пользователя (автоматически предоставляется зависимостью).

### Ответ:
В случае успешной аутентификации возвращается JSON-объект с данными пользователя и токенами доступа.

Данный запрос проблематично тестировать через Postman.


### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/yandex_user_info" -H "Authorization: YOUR_YANDEX_OAUTH_TOKEN"

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка при обращении к Yandex API",<br/>
 "code": 500,<br/>
 "type": "server_error"<br/>
}


### Реализация API:
```
@router.post("/yandex_user_info")
async def yandex_user_info(response_for_cookie: Response,
                          oauth_token: str =
                          Header(None, title="OAuth-токен"),
                          db: AsyncSession = Depends(get_db), x_fingerprint=Depends(get_fingerprint)):
   # URL для отправки запроса
   url = "https://login.yandex.ru/info"
   # параметры запроса на основе переданных значений
   params = {
       "format": "json",
       "jwt_secret": Auth.secret,
   }
…
return result
else:
   raise HTTPException(status_code=500, detail=f"Ошибка при обращении к Yandex API: {response}")
```
## 9. Данный запрос позволяет пользователю зарегистрироваться с помощью аккаунта Вконтакте 
### Метод: 
POST
### Путь: 
/vk_user_info
### Параметры запроса:
 - vk: Объект, содержащий информацию о пользователе от VK.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - x_fingerprint: Информация о браузере пользователя (автоматически предоставляется зависимостью).

### Ответ:
В случае успешной аутентификации возвращается JSON-объект с данными пользователя и токенами доступа.

Данный запрос проблематично тестировать через Postman.


### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/vk_user_info" -d '{"silent_token": "YOUR_SILENT_TOKEN", "uuid": "YOUR_UUID"}'

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка при обращении к VK API",<br/>
 "code": 500,<br/>
 "type": "server_error"<br/>
}


### Реализация API:
```
@router.post("/vk_user_info")
async def vk_user_info(response_for_cookie: Response, vk: users.VkUserInfo,  # vk_data: users.VkExchangeSilentAuthToken,
                      db: AsyncSession = Depends(get_db), x_fingerprint=Depends(get_fingerprint)):
   # URL для отправки запроса
   url = "https://api.vk.com/method/auth.exchangeSilentAuthToken"
...
return authorize_data
else:
   raise HTTPException(status_code=500, detail=f"Ошибка при обращении к VK API: {vk_response_data}")
```

## 10. Данный запрос позволяет авторизовать пользователя, зарегистрированного с помощью Яндекс или Вконтакте  
### Метод: 
POST
### Путь: 
/sign-in.external
### Параметры запроса:
 - response_for_cookie: Объект ответа, используется для установки cookie с токенами доступа.
 - user: Объект, содержащий данные для входа.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - x_fingerprint: Информация о браузере пользователя (автоматически предоставляется зависимостью).


### Ответ:
В случае успешной аутентификации возвращается JSON-объект с данными пользователя и токенами доступа.

Данный запрос проблематично тестировать через Postman.


### Пример использования:
curl -X POST <br/>
"https://backend-test.diaclass.kz/sign-in.external" -d '{"email": "user@example.com", "vk": "1234567890"}'

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Forbidden",<br/>
 "code": 403,<br/>
 "type": "forbidden_error"<br/>
}


### Реализация API:
```
@router.post('/sign-in.external')
async def external_login(response_for_cookie: Response,
                        user: users.ExternalUserLogin,
                        db: AsyncSession = Depends(get_db),
                        x_fingerprint=Depends(get_fingerprint)):
   if not (user.vk or user.yandex):
       raise HTTPException(status_code=403, detail="Forbidden")
...
return {'user': {**jsonable_encoder(users.UserAuth(**jsonable_encoder(db_user)))}, 'access_token': access_token,
       "expires_in": expires_in}
```

# PATCH запросы
## 11. Данный запрос изменяет название выбранной презентации
### Метод: 
PATCH
### Путь: 
/presentation/rename
### Параметры запроса:
 - presentation: Объект, содержащий идентификатор и новое название презентации.
 - name: Новое название презентации (строка, максимум 100 символов, разрешены буквы, цифры и некоторые спецсимволы).
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Текущий аутентифицированный пользователь (автоматически предоставляется зависимостью).

### Ответ:
В случае успешного запроса, сервер возвращает код статуса HTTP 200 (OK) и обновленную информацию о презентации.

![image_2024-02-04_22-51-34](https://github.com/AnaSKBK/PAPS/assets/128895913/60296e35-6ba4-44a8-9954-ab1aaba8b39c)


### Пример использования:
curl -X PATCH  <br/>
"https://backend-test.diaclass.kz/presentation/rename" -H "Authorization: Bearer YOUR_ACCESS_TOKEN" -d '{"id": 1, "name": "Новое название презентации"}'


### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка проверки администратора",<br/>
 "code": 403,<br/>
 "type": "forbidden_error"<br/>
}


### Реализация API:
```
@router.patch("/presentation/rename", status_code=200)
async def rename_presentation(
       presentation: presentation_schemas.PresentationRename,
       name: str = Query(max_length=100, regex="[a-zA-Zа-яА-Я0-9_()-*""₽€£¥$+=:;,?!]"),
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, presentation_id=presentation.id)
   presentation = await presentation_requests.rename_presentation(db, presentation.id, name)
   return presentation
```
## 12. Данный запрос позволяет пользователю изменить тип слайда
### Метод: 
PATCH
### Путь: 
/presentation/slide/change-type
### Параметры запроса:
 - slide: Объект, содержащий информацию о слайде и новом типе слайда.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Текущий аутентифицированный пользователь (автоматически предоставляется зависимостью).

### Ответ:
В случае успешного изменения типа слайда возвращается обновленный объект слайда.

![image_2024-02-04_22-52-07](https://github.com/AnaSKBK/PAPS/assets/128895913/177b4884-8eba-4b68-92a8-1f5fdff94fad)


### Пример использования:
curl -X PATCH  <br/>
"https://backend-test.diaclass.kz/presentation/slide/change-type" -d '{"id": 123, "id_slide_type": 8}'


### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Forbidden",<br/>
 "code": 403,<br/>
 "type": "forbidden_error"<br/>
}


### Реализация API:
```
@router.patch("/presentation/slide/change-type", status_code=200)
async def select_slide_type(
       slide: slide_schemas.SlideSetType,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, slide_id=slide.id)
   res = await presentation_requests.select_slide_type(db, slide.id, slide.id_slide_type, current_user)
   return res
```
# DELETE запросы
## 13. Данный запрос удаляет выбранный слайд
### Метод: 
DELETE
### Путь: 
 /presentation/slide/delete
### Параметры запроса:
 - slide: Объект, содержащий информацию об удаляемом слайде.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Текущий аутентифицированный пользователь (автоматически предоставляется зависимостью).

### Ответ:
В случае успешного запроса, сервер возвращает код статуса HTTP 200 (OK) и информацию о удаленном слайде.

![image_2024-02-04_22-51-47](https://github.com/AnaSKBK/PAPS/assets/128895913/538808b8-9c15-4bdd-916a-b2c8b5a01f08)



### Пример использования:
curl -X DELETE   <br/>
"https://backend-test.diaclass.kz/presentation/slide/delete" -H "Authorization: Bearer YOUR_ACCESS_TOKEN" -d '{"id": 1, "id_slide_type": 8}'

### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка проверки администратора",<br/>
 "code": 403,<br/>
 "type": "forbidden_error"<br/>
}


### Реализация API:
```
@router.delete("/presentation/slide/delete", status_code=200)
async def del_slide(
       slide: slide_schemas.SlideDelete,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, slide_id=slide.id)
   if slide.id_slide_type == 8:
       await del_image_from_slide(db, slide.id)
   slide = await presentation_requests.del_slide(db, slide.id)
   return slide

```
## 14. Данный запрос удаляет выбранную презентацию
### Метод: 
DELETE
### Путь: 
/presentation/delete
### Параметры запроса:
 - presentation: Объект, содержащий идентификатор удаляемой презентации.
 - db: Сессия базы данных (автоматически предоставляется зависимостью).
 - current_user: Текущий аутентифицированный пользователь (автоматически предоставляется зависимостью).

### Ответ:
В случае успешного запроса, сервер возвращает код статуса HTTP 200 (OK).


![image_2024-02-04_22-51-27](https://github.com/AnaSKBK/PAPS/assets/128895913/9042e80d-09d0-4952-96c5-b5bdffc8007f)


### Пример использования:
curl -X DELETE   <br/>
"https://backend-test.diaclass.kz/presentation/delete" -H "Authorization: Bearer YOUR_ACCESS_TOKEN" -d '{"id": 1}'


### Ошибки:
В случае возникновения ошибки, сервер возвращает соответствующий HTTP-статус и информацию об ошибке в формате JSON.<br/>
Пример ошибки:<br/>
{<br/>
 "detail": "Ошибка проверки администратора",<br/>
 "code": 403,<br/>
 "type": "forbidden_error"<br/>
}


### Реализация API:
```
@router.delete("/presentation/delete", status_code=200)
async def delete_presentation(
       presentation: presentation_schemas.PresentationDelete,
       db: AsyncSession = Depends(get_db),
       current_user=Depends(user_requests.get_current_user_by_auth)):
   await presentation_requests.check_admin(db, current_user, presentation_list_id=presentation.id)
   await del_image_from_presentation(db, presentation_ids=presentation.id)
   presentation = await presentation_requests.delete_presentation(db, presentation.id)
   return presentation

```
