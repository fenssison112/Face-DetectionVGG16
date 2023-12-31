Данный код отвечает за распознавания лица в режиме реального времени. Благодаря аугментации данных можно обучить модель распознавать лицо с небольшим количеством данных. Также можно обучить модель распознавать не только лицо, можно обучить распознавать авто, номерные знаки и т.д.

1. Импортирование нужных бибилиотек

        import tensorflow as tf
        import json
        import numpy as np
        from matplotlib import pyplot as plt
        import os
        import time
        import uuid
        import cv2
        import albumentations as alb
        from keras.models import Model
        from keras.layers import Input, Conv2D, Dense, GlobalMaxPooling2D
        from keras.applications import VGG16
        from keras.models import load_model

2. Создаем свой датасет с помощью OpenCV

        IMAGES_PATH = os.path.join('data','images')
        number_images = 30

        cap = cv2.VideoCapture(1)
        for imgnum in range(number_images):
            print('Collecting image {}'.format(imgnum))
            ret, frame = cap.read()
            imgname = os.path.join(IMAGES_PATH,f'{str(uuid.uuid1())}.jpg')
            cv2.imwrite(imgname, frame)
            cv2.imshow('frame', frame)
            time.sleep(0.5)
        
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break
        cap.release()
        cv2.destroyAllWindows()

3. Запускаем Labelme

        labelme

После запуска labelme, выбираем путь к папке с изображениями 
1. Нажимаем Open Dir
2. Далее чтобы указать путь к папке куда будут сохраняться координаты нажимаем File -> Change Output Dir и указываем папку
3. Нажимаем Edit -> Create Rectangle и выделяем нужный объект 2 точками, и именуем его
4. Далее просто выделяем нужный объект, пока не закончатся фото, там где нет нужного объекта, ничего не выделяем и пропускаем изображение

![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/3d3c9bb5-4127-4336-90a1-3e3b227b24c9)

4. Загружаем изображение в конвейер передачи данных TF
   
        images = tf.data.Dataset.list_files('/путь к папке с изображениями/*.jpg')
        images.as_numpy_iterator().next()
        def load_image(x):
            byte_img = tf.io.read_file(x)
            img = tf.io.decode_jpeg(byte_img)
            return img
        
        images = images.map(load_image)
        images.as_numpy_iterator().next()
        type(images)
   
5. Просмотр изображений с помощью Matplotlib

        image_generator = images.batch(4).as_numpy_iterator()
        plot_images = image_generator.next()
        
        fig, ax = plt.subplots(ncols=4, figsize=(20,20))
        for idx, image in enumerate(plot_images):
            ax[idx].imshow(image)
        plt.show()
   ![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/4b752e70-46b4-4b0e-8877-e93fa42d2d27)

6. ВРУЧНУЮ РАСПРЕДЕЛИТЕ ДАННЫЕ TRAIN TEST И VAL

Для этого в корневой папке (data) нужно создать 3 папки train, test и val. Также в каждой из этих папок нужно создать еще 2 папки images и labels
Далее вручную распределите изображения в каждую из папок. Например имея 90 изображений, 63 из них нужно поместить в папку images в train, 13 изображений поместить в папку images в test и 14 поместить в папку images в val

        data_path = '/путь к корневой папке(data)/'  
        for folder in ['train', 'test', 'val']:
            for file in os.listdir(os.path.join(data_path, folder, 'images')):
                filename = file.split('.')[0] + '.json'
                existing_filepath = os.path.join(data_path, 'labels', filename)
                if os.path.exists(existing_filepath):
                    new_filepath = os.path.join(data_path, folder, 'labels', filename)
                    os.replace(existing_filepath, new_filepath)
7.Настройка конвейера преобразования альбомов

Здесь ппроисходит аугментация изображений. Указывается процентное соотношение. Подробней смотри на сайте https://albumentations.ai/docs/getting_started/bounding_boxes_augmentation/

    augmentor = alb.Compose([alb.RandomCrop(width=450, height=450),
                            alb.HorizontalFlip(p=0.5),
                            alb.RandomBrightnessContrast(p=0.2),
                            alb.RandomGamma(p=0.2),
                            alb.RGBShift(p=0.2),
                            alb.VerticalFlip(p=0.5)],
                            bbox_params=alb.BboxParams(format='albumentations',
                                                      label_fields=['class_labels']))
8. Загрузите тестовое изображение и аннотацию с помощью OpenCV и JSON

            img = cv2.imread(os.path.join('/data/train/images/2c00f069-f974-11ed-9efb-2a024c400e1c.jpg'))
            with open(os.path.join('/data/train/labels/2c00f069-f974-11ed-9efb-2a024c400e1c.json'), 'r') as f:
                label = json.load(f)
                
            label['shapes'][0]['points']

9. Извлеките координаты и измените масштаб в соответствии с разрешением изображения
    
            coords = [0,0,0,0]
            coords[0] = label['shapes'][0]['points'][0][0]
            coords[1] = label['shapes'][0]['points'][0][1]
            coords[2] = label['shapes'][0]['points'][1][0]
            coords[3] = label['shapes'][0]['points'][1][1]
            
            coords = list(np.divide(coords, [640,480,640,480]))
            coords

10. Применяем аугментации и просмотр результатов

            augmented = augmentor(image=img, bboxes=[coords], class_labels=['face'])
            augmented['bboxes'][0][2:]
            augmented['bboxes'][0][2:]
            
            cv2.rectangle(augmented['image'],
                          tuple(np.multiply(augmented['bboxes'][0][:2], [450,450]).astype(int)),
                          tuple(np.multiply(augmented['bboxes'][0][2:], [450,450]).astype(int)),
                                (255,0,0), 2)
            
            plt.imshow(augmented['image'])
    ![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/8c73dcfe-f062-4fe6-8d9a-da7ff84aab90)

    

12. Создаем папку с аугментациями и запускаем конвейер

            auf_data = '/content/drive/MyDrive/aug_data'
            for partition in ['train','test','val']:
                for image in os.listdir(os.path.join(data_path, partition, 'images')):
                    img = cv2.imread(os.path.join(data_path, partition, 'images', image))
    
            coords = [0,0,0.00001,0.00001]
            label_path = os.path.join(data_path, partition, 'labels', f'{image.split(".")[0]}.json')
            if os.path.exists(label_path):
                with open(label_path, 'r') as f:
                    label = json.load(f)
    
                coords[0] = label['shapes'][0]['points'][0][0]
                coords[1] = label['shapes'][0]['points'][0][1]
                coords[2] = label['shapes'][0]['points'][1][0]
                coords[3] = label['shapes'][0]['points'][1][1]
                coords = list(np.divide(coords, [640,480,640,480]))
    
            try:
                for x in range(60):
                    augmented = augmentor(image=img, bboxes=[coords], class_labels=['face'])
                    cv2.imwrite(os.path.join(auf_data, partition, 'images', f'{image.split(".")[0]}.{x}.jpg'), augmented['image'])
    
                    annotation = {}
                    annotation['image'] = image
    
                    if os.path.exists(label_path):
                        if len(augmented['bboxes']) == 0:
                            annotation['bbox'] = [0,0,0,0]
                            annotation['class'] = 0
                        else:
                            annotation['bbox'] = augmented['bboxes'][0]
                            annotation['class'] = 1
                    else:
                        annotation['bbox'] = [0,0,0,0]
                        annotation['class'] = 0
    
    
                    with open(os.path.join(auf_data, partition, 'labels', f'{image.split(".")[0]}.{x}.json'), 'w') as f:
                        json.dump(annotation, f)
    
            except Exception as e:
                print(e)

13. Загружаем аугментированные изображения в набор данных Tensorflow

            train_images = tf.data.Dataset.list_files('/content/drive/MyDrive/aug_data/train/images/*.jpg', shuffle=False)
            train_images = train_images.map(load_image)
            train_images = train_images.map(lambda x: tf.image.resize(x, (224,224)))
            train_images = train_images.map(lambda x: x/255)
            
            test_images = tf.data.Dataset.list_files('/content/drive/MyDrive/aug_data/test/images/*.jpg', shuffle=False)
            test_images = test_images.map(load_image)
            test_images = test_images.map(lambda x: tf.image.resize(x, (224,224)))
            test_images = test_images.map(lambda x: x/255)
            
            val_images = tf.data.Dataset.list_files('/content/drive/MyDrive/aug_data/val/images/*.jpg', shuffle=False)
            val_images = val_images.map(load_image)
            val_images = val_images.map(lambda x: tf.image.resize(x, (224,224)))
            val_images = val_images.map(lambda x: x/255)
            
            train_images.as_numpy_iterator().next()
    
14. Функция загрузки меток
    
            def load_labels(label_path):
                with open(label_path.numpy(), 'r', encoding = "utf-8") as f:
                    label = json.load(f)
            return [label['class']], label['bbox']

15. Загружаем метки в набор данных Tensorflow
    
            train_labels = tf.data.Dataset.list_files('/content/drive/MyDrive/aug_data/train/labels/*.json', shuffle=False)
            train_labels = train_labels.map(lambda x: tf.py_function(load_labels, [x], [tf.uint8, tf.float16]))
            
            test_labels = tf.data.Dataset.list_files('/content/drive/MyDrive/aug_data/test/labels/*.json', shuffle=False)
            test_labels = test_labels.map(lambda x: tf.py_function(load_labels, [x], [tf.uint8, tf.float16]))
            
            val_labels = tf.data.Dataset.list_files('/content/drive/MyDrive/aug_data/val/labels/*.json', shuffle=False)
            val_labels = val_labels.map(lambda x: tf.py_function(load_labels, [x], [tf.uint8, tf.float16]))
            
            train_labels.as_numpy_iterator().next()

16. Проверка длинны разделов
    
            len(train_images), len(train_labels), len(test_images), len(test_labels), len(val_images), len(val_labels)
    
17. Создание окончательных наборов данных (изображений/меток)

            train = tf.data.Dataset.zip((train_images, train_labels))
            train = train.shuffle(3000)
            train = train.batch(8)
            train = train.prefetch(4)
            
            test = tf.data.Dataset.zip((test_images, test_labels))
            test = test.shuffle(1300)
            test = test.batch(8)
            test = test.prefetch(4)
            
            val = tf.data.Dataset.zip((val_images, val_labels))
            val = val.shuffle(1000)
            val = val.batch(8)
            val = val.prefetch(4)
    
            train.as_numpy_iterator().next()[1]

18. Просмотр изображений и аннотаций

            data_samples = train.as_numpy_iterator()
            res = data_samples.next()
            fig, ax = plt.subplots(ncols=4, figsize=(20,20))
            for idx in range(4):
                sample_image = res[0][idx]
                sample_coords = res[1][1][idx]
    
            cv2.rectangle(sample_image,
                      tuple(np.multiply(sample_coords[:2], [224,224]).astype(int)),
                      tuple(np.multiply(sample_coords[2:], [224,224]).astype(int)),
                            (255,0,0), 2)
    
            ax[idx].imshow(sample_image)
    ![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/517ed5f5-602d-4f19-9152-bf2a625f6637)


20. Скачать VGG16

            vgg = VGG16(include_top=False)
            vgg.summary()

21. Создаем архитектуру VGG16

            def build_model(): 
            input_layer = Input(shape=(120, 120, 3))
        
            conv1 = Conv2D(64, (3, 3), activation='relu', padding='same', name='block1_conv1')(input_layer)
            conv2 = Conv2D(64, (3, 3), activation='relu', padding='same', name='block1_conv2')(conv1)
            pool1 = MaxPooling2D((2, 2), strides=(2, 2), name='block1_pool')(conv2)
            
            conv3 = Conv2D(128, (3, 3), activation='relu', padding='same', name='block2_conv1')(pool1)
            conv4 = Conv2D(128, (3, 3), activation='relu', padding='same', name='block2_conv2')(conv3)
            pool2 = MaxPooling2D((2, 2), strides=(2, 2), name='block2_pool')(conv4)
            
            conv5 = Conv2D(256, (3, 3), activation='relu', padding='same', name='block3_conv1')(pool2)
            conv6 = Conv2D(256, (3, 3), activation='relu', padding='same', name='block3_conv2')(conv5)
            conv7 = Conv2D(256, (3, 3), activation='relu', padding='same', name='block3_conv3')(conv6)
            pool3 = MaxPooling2D((2, 2), strides=(2, 2), name='block3_pool')(conv7)
            
            conv8 = Conv2D(512, (3, 3), activation='relu', padding='same', name='block4_conv1')(pool3)
            conv9 = Conv2D(512, (3, 3), activation='relu', padding='same', name='block4_conv2')(conv8)
            conv10 = Conv2D(512, (3, 3), activation='relu', padding='same', name='block4_conv3')(conv9)
            pool4 = MaxPooling2D((2, 2), strides=(2, 2), name='block4_pool')(conv10)
        
            f1 = GlobalMaxPooling2D()(pool4)
            class1 = Dense(2048, activation='relu', name='class1')(f1)
            class2 = Dense(1, activation='sigmoid', name='class2')(class1)
            regress1 = Dense(2048, activation='relu', name='regress1')(f1)
            regress2 = Dense(4, activation='sigmoid', name='regress2')(regress1)
        
            facetracker = Model(inputs=input_layer, outputs=[class2, regress2])
        
            return facetracker

22. Тестируем нейронную сеть

            facetracker = build_model()
            facetracker.summary()
            
            X, y = train.as_numpy_iterator().next()
            X.shape
            classes, coords = facetracker.predict(X)
            classes, coords

23. Определите оптимизатор и LR

        batches_per_epoch = len(train)
        lr_decay = (1./0.75 -1)/batches_per_epoch
        
        opt = tf.keras.optimizers.Adam(learning_rate=0.0001)
        def localization_loss(y_true, yhat):
        delta_coord = tf.reduce_sum(tf.square(y_true[:,:2] - yhat[:,:2]))
        
        h_true = y_true[:,3] - y_true[:,1]
        w_true = y_true[:,2] - y_true[:,0]
        
        h_pred = yhat[:,3] - yhat[:,1]
        w_pred = yhat[:,2] - yhat[:,0]
        
        delta_size = tf.reduce_sum(tf.square(w_true - w_pred) + tf.square(h_true-h_pred))
        
        return delta_coord + delta_size
        
        classloss = tf.keras.losses.BinaryCrossentropy()
        regressloss = localization_loss


24. Проверяем показатели потерь

            localization_loss(y[1], coords)
            classloss(y[0], classes)
            regressloss(y[1], coords)
    
25. Тренируем нейронную сеть


            class FaceTracker(Model): 
            def __init__(self, eyetracker,  **kwargs): 
                super().__init__(**kwargs)
                self.model = eyetracker
        
            def compile(self, opt, classloss, localizationloss, **kwargs):
                super().compile(**kwargs)
                self.closs = classloss
                self.lloss = localizationloss
                self.opt = opt
            
            def train_step(self, batch, **kwargs): 
                
                X, y = batch
                
                with tf.GradientTape() as tape: 
                    classes, coords = self.model(X, training=True)
                    
                    batch_classloss = self.closs(y[0], classes)
                    batch_localizationloss = self.lloss(tf.cast(y[1], tf.float32), coords)
                    
                    total_loss = batch_localizationloss+0.5*batch_classloss
                    
                    grad = tape.gradient(total_loss, self.model.trainable_variables)
                
                opt.apply_gradients(zip(grad, self.model.trainable_variables))
                
                return {"total_loss":total_loss, "class_loss":batch_classloss, "regress_loss":batch_localizationloss}
            
            def test_step(self, batch, **kwargs): 
                X, y = batch
                
                classes, coords = self.model(X, training=False)
                
                batch_classloss = self.closs(y[0], classes)
                batch_localizationloss = self.lloss(tf.cast(y[1], tf.float32), coords)
                total_loss = batch_localizationloss+0.5*batch_classloss
                
                return {"total_loss":total_loss, "class_loss":batch_classloss, "regress_loss":batch_localizationloss}
                
            def call(self, X, **kwargs): 
                return self.model(X, **kwargs)

            model = FaceTracker(facetracker)
            model.compile(opt, classloss, regressloss)
            logdir='/content/drive/MyDrive/logs'
            tensorboard_callback = tf.keras.callbacks.TensorBoard(log_dir=logdir)
            
            
            hist = model.fit(train, epochs=15, validation_data=val, callbacks=[tensorboard_callback])
            hist.history
            
            fig, ax = plt.subplots(ncols=3, figsize=(20,5))
            
            ax[0].plot(hist.history['total_loss'], color='teal', label='loss')
            ax[0].plot(hist.history['val_total_loss'], color='orange', label='val loss')
            ax[0].title.set_text('Loss')
            ax[0].legend()
            
            ax[1].plot(hist.history['class_loss'], color='teal', label='class loss')
            ax[1].plot(hist.history['val_class_loss'], color='orange', label='val class loss')
            ax[1].title.set_text('Classification Loss')
            ax[1].legend()
            
            ax[2].plot(hist.history['regress_loss'], color='teal', label='regress loss')
            ax[2].plot(hist.history['val_regress_loss'], color='orange', label='val regress loss')
            ax[2].title.set_text('Regression Loss')
            ax[2].legend()
            
            plt.show()
    ![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/83a5b184-0c5f-4157-a017-ef1a96c254d6)

            
            test_data = test.as_numpy_iterator()
            test_sample = test_data.next()
            yhat = facetracker.predict(test_sample[0])
            fig, ax = plt.subplots(ncols=4, figsize=(20,20))
            for idx in range(4):
                sample_image = test_sample[0][idx]
                sample_coords = yhat[1][idx]
            
                if yhat[0][idx] > 0.9:
                    cv2.rectangle(sample_image,
                                  tuple(np.multiply(sample_coords[:2], [224,224]).astype(int)),
                                  tuple(np.multiply(sample_coords[2:], [224,224]).astype(int)),
                                        (255,0,0), 2)
            
                ax[idx].imshow(sample_image)
            ![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/c328c18b-f4b1-442f-9892-b01a5b7d40bd)

                
    
27. Сохраняем модель
    
            facetracker.save('facetracker-15.h5')
    
28. Проверяем модель в реальном времени
    
            cap = cv2.VideoCapture(1)
            while cap.isOpened():
                _ , frame = cap.read()
                frame = frame[50:500, 50:500,:]
                
                rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
                resized = tf.image.resize(rgb, (224,224))
                
                yhat = facetracker.predict(np.expand_dims(resized/255,0))
                sample_coords = yhat[1][0]
                
                if yhat[0] > 0.5: 
                    # Controls the main rectangle
                    cv2.rectangle(frame, 
                                  tuple(np.multiply(sample_coords[:2], [450,450]).astype(int)),
                                  tuple(np.multiply(sample_coords[2:], [450,450]).astype(int)), 
                                        (255,0,0), 2)
                    # Controls the label rectangle
                    cv2.rectangle(frame, 
                                  tuple(np.add(np.multiply(sample_coords[:2], [450,450]).astype(int), 
                                                [0,-30])),
                                  tuple(np.add(np.multiply(sample_coords[:2], [450,450]).astype(int),
                                                [80,0])), 
                                        (255,0,0), -1)
                    
                    # Controls the text rendered
                    cv2.putText(frame, 'face', tuple(np.add(np.multiply(sample_coords[:2], [450,450]).astype(int),
                                                           [0,-5])),
                                cv2.FONT_HERSHEY_SIMPLEX, 1, (255,255,255), 2, cv2.LINE_AA)
                
                cv2.imshow('EyeTrack', frame)
                
                if cv2.waitKey(1) & 0xFF == ord('q'):
                    break
            cap.release()
            cv2.destroyAllWindows()
    ![image](https://github.com/fenssison112/Face-DetectionVGG16/assets/109478845/40311d70-4c10-4db9-affc-36f306af6677)

    




                        
                                                  
