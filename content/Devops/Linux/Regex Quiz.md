❓ **Питання 1:** How can you search for a pattern in multiple files using grep (including subdirectories)?

|Варіант|Команда|Пояснення|
|---|---|---|
|✅ a|`grep -r "pattern" file1.txt file2.txt`|✅ Рекурсивний пошук у файлах і підкаталогах. Знайде шаблон у всіх файлах.|
|❌ b|`grep -v "pattern" file1.txt file2.txt`|❌ Виводить рядки, **які не відповідають** шаблону. Не для пошуку збігів.|
|❌ c|`grep -l "pattern" file1.txt file2.txt`|❌ Показує лише імена файлів, де є збіг — без вмісту. Не рекурсивно.|
|❌ d|`grep -c "pattern" file1.txt file2.txt`|❌ Показує лише **кількість** входжень у кожному файлі.|
|❌ e|`grep -o "pattern" file1.txt file2.txt`|❌ Показує тільки ті частини рядка, що збіглися. Не шукає рекурсивно.|

---

❓ **Питання 2:** How can you search for a specific word in a file using grep?

|Варіант|Команда|Пояснення|
|---|---|---|
|❌ a|`grep -c "word" file.txt`|❌ Показує лише кількість входжень, а не самі рядки.|
|✅ b|`grep -w "word" file.txt`|✅ Шукає **повне слово** "word", не як частину інших слів.|
|❌ c|`grep -r "word" file.txt`|❌ Рекурсивний пошук — не актуально для одного файлу.|
|❌ d|`grep -l "word" file.txt`|❌ Показує тільки ім'я файлу, де є слово.|
|❌ e|`grep -v "word" file.txt`|❌ Показує ті рядки, які **не містять** слово "word".|
❓ **Питання 3:**  
What does the regex pattern `"[0-9]+"` match?

|Варіант|Команда|Пояснення|
|---|---|---|
|❌ a|Any special character|❌ Спеціальні символи не входять до діапазону `0-9`.|
|✅ b|Any digit between 0 and 9|✅ Шаблон `[0-9]+` означає **одну або більше цифр** від 0 до 9.|
|❌ c|Any uppercase letter|❌ Для великих літер використовують `[A-Z]`.|
|❌ d|Any lowercase letter|❌ Для малих літер використовують `[a-z]`.|
❓ **Питання 4:**  
What does the regex pattern `"\d{2,4}"` match?

|Варіант|Команда|Пояснення|
|---|---|---|
|❌ a|Any three-digit number|❌ Не тільки три цифри, а **від двох до чотирьох**.|
|✅ b|Any digit character repeated between two and four times|✅ `\d{2,4}` означає **цифра**, повторена **від 2 до 4 разів**.|
|❌ c|Any non-digit character|❌ Для цього використовують `\D`, не `\d`.|
|❌ d|Any two-digit number|❌ Включає також 3- та 4-цифрові числа.|
|❌ e|Any four-digit number|❌ Включає також 2- та 3-цифрові варіанти.|
❓ **Питання 5:**  
What is the purpose of the `sed` command in Linux?

|Варіант|Команда|Пояснення|
|---|---|---|
|❌ a|To extract specific columns from a CSV file|❌ Це більше про `cut` або `awk`, а не `sed`.|
|❌ b|To compress files into a single archive|❌ Це робить `tar`, не `sed`.|
|✅ c|To manipulate text streams and files|✅ `sed` (stream editor) — інструмент для **редагування тексту в потоці**.|
|❌ d|To create and manage user accounts|❌ Для цього є команди типу `useradd`, `usermod`.|
|❌ e|To search for files within a directory|❌ Це функція `find`, не `sed`.|
