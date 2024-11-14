<!DOCTYPE html>
<html lang="ar">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>مراقبة سلوك طفل توحد باستخدام الكاميرا</title>
    <style>
        body {
            font-family: 'Cairo', sans-serif;
            text-align: center;
            background-color: #f2f2f2;
            margin: 0;
            padding: 20px;
        }
        #video {
            border: 1px solid #ccc;
            margin-top: 20px;
            width: 80%;
            max-width: 600px;
            border-radius: 8px;
        }
        .result {
            font-size: 20px;
            margin-top: 20px;
        }
        .feedback {
            font-size: 18px;
            color: green;
            margin-top: 10px;
        }
        .footer {
            margin-top: 50px;
            font-size: 14px;
            color: #888;
        }
        button {
            background-color: #4CAF50;
            color: white;
            padding: 10px 20px;
            font-size: 18px;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            margin-top: 20px;
        }
        button:hover {
            background-color: #45a049;
        }
        #report {
            margin-top: 30px;
            font-size: 18px;
            padding: 10px;
            border-radius: 8px;
            border: 1px solid #ccc;
            background-color: #f9f9f9;
            display: none; /* إخفاء التقرير في البداية */
        }
    </style>
</head>
<body>

    <h1>مراقبة سلوك طفل توحد باستخدام الكاميرا</h1>
    <p>يرجى السماح بالوصول إلى الكاميرا لتسجيل سلوكيات الطفل خلال الجلسة.</p>

    <!-- زر تشغيل الكاميرا -->
    <button id="startButton">تشغيل الكاميرا</button>

    <!-- الفيديو المعروض من الكاميرا -->
    <video id="video" width="640" height="480" autoplay></video>

    <div class="result" id="result"></div>
    <div class="feedback" id="feedback"></div>

    <!-- تقرير السلوكيات -->
    <div id="report">
        <h3>تقرير السلوكيات:</h3>
        <p id="behaviorReport"></p>
    </div>

    <!-- إضافة بيانات "تم الإنشاء بواسطة" في أسفل الصفحة -->
    <div class="footer">
        <p>أنشئ بواسطة: <strong>الطالب لؤي الخطيب</strong></p>
        <p>الدكتور المشرف: <strong>د. بيان عز الدين الدقامسه</strong></p>
        <p>جامعة جدارا</p>
    </div>

    <!-- تحميل المكتبة الخاصة بالتعرف على الوجه -->
    <script defer src="https://cdn.jsdelivr.net/npm/face-api.js"></script>

    <script>
        async function startCamera() {
            // تحميل النماذج الخاصة بـ Face-api.js
            await faceapi.nets.ssdMobilenetv1.loadFromUri('/models');
            await faceapi.nets.faceLandmark68Net.loadFromUri('/models');
            await faceapi.nets.faceRecognitionNet.loadFromUri('/models');

            // الوصول إلى الكاميرا الأمامية
            const stream = await navigator.mediaDevices.getUserMedia({
                video: true
            });

            const videoElement = document.getElementById('video');
            videoElement.srcObject = stream;

            videoElement.onplay = () => {
                detectFace(videoElement);
            };
        }

        async function detectFace(videoElement) {
            const detections = await faceapi.detectAllFaces(videoElement)
                .withFaceLandmarks()
                .withFaceDescriptors();

            // تقديم الملاحظات بناءً على كشف الوجه
            if (detections.length > 0) {
                const expression = analyzeExpression(detections);
                document.getElementById('result').innerText = 'تم الكشف عن وجه الطفل، يحاول التفاعل.';
                document.getElementById('feedback').innerText = expression;
            } else {
                document.getElementById('result').innerText = 'لم يتم الكشف عن وجه الطفل. تأكد من أن الطفل في الإطار.';
                document.getElementById('feedback').innerText = '';
            }

            // إعادة المحاولة
            setTimeout(() => detectFace(videoElement), 100);
        }

        function analyzeExpression(detections) {
            let feedback = '';
            let behaviors = [];

            detections.forEach(detection => {
                const landmarks = detection.landmarks;
                const mouth = landmarks.getMouth();
                const eyeLeft = landmarks.getLeftEye();
                const eyeRight = landmarks.getRightEye();

                // التحقق من الابتسامة
                if (isSmiling(mouth)) {
                    feedback += 'الطفل يظهر ابتسامة، تفاعل إيجابي.\n';
                    behaviors.push('ابتسامة');
                } 
                // التحقق من العيون المغلقة
                else if (areEyesClosed(eyeLeft, eyeRight)) {
                    feedback += 'الطفل يغلق عينيه، قد يكون في حالة إغفال أو غير متفاعل.\n';
                    behaviors.push('غلق العيون');
                } 
                // التحقق من الاستجابة للتفاعل
                else if (isLookingAway(eyeLeft, eyeRight)) {
                    feedback += 'الطفل ينظر بعيدًا عن الكاميرا، قد يكون مشتت الانتباه.\n';
                    behaviors.push('النظر بعيدًا');
                }
                // السلوكيات النمطية مثل عدم الحركة
                else if (isStill()) {
                    feedback += 'الطفل لا يتحرك كثيرًا، قد يكون في حالة ثابتة أو سلوك نمطي.\n';
                    behaviors.push('ثبات وعدم حركة');
                } 
                // التفاعل الحيادي
                else {
                    feedback += 'الطفل يبدو في حالة تفاعل حيادي.\n';
                    behaviors.push('تفاعل حيادي');
                }
            });

            // عرض السلوكيات في التقرير
            document.getElementById('behaviorReport').innerText = 'السلوكيات التي تم ملاحظتها: ' + behaviors.join(', ');

            return feedback;
        }

        function isSmiling(mouth) {
            const mouthWidth = mouth[12].x - mouth[6].x;
            const mouthHeight = mouth[3].y - mouth[9].y;

            // نسبة العرض إلى الارتفاع تشير إلى ابتسامة
            return mouthWidth > 1.5 * mouthHeight;
        }

        function areEyesClosed(leftEye, rightEye) {
            const leftEyeWidth = leftEye[3].x - leftEye[0].x;
            const rightEyeWidth = rightEye[3].x - rightEye[0].x;

            // إذا كانت العيون مغلقة أو ضيقة
            return leftEyeWidth < 30 && rightEyeWidth < 30;
        }

        function isLookingAway(leftEye, rightEye) {
            // التحقق من حركة العينين بعيدًا عن الكاميرا
            return leftEye[0].x < 40 || rightEye[3].x > 600;
        }

        function isStill() {
            // هنا نحدد ما إذا كان الطفل ثابتًا أم لا
            return true;  // نعود بالقيمة الحقيقية، يمكن تحديد معايير أكثر دقة
        }

        // إضافة تأخير لتوليد التقرير بعد 10 ثوانٍ
        function generateReport() {
            setTimeout(function() {
                document.getElementById('report').style.display = 'block'; // عرض التقرير بعد 10 ثوانٍ
            }, 10000); // 10 ثوانٍ من تشغيل الكاميرا
        }

        // عندما يتم الضغط على الزر، يبدأ تشغيل الكاميرا ويظهر التقرير
        document.getElementById('startButton').addEventListener('click', () => {
            startCamera();
            generateReport();
        });
    </script>

</body>
</html>
