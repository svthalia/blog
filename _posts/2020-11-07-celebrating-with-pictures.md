---
layout: post
title: Celebrating a birthday with pictures
author: sébastiaan
---

As a Study Assocation you organise an insane amount of events in a lecture year. At Thalia we have a committee that takes photos during those events. And they have really been doing their job in the past 5 years.

Since we are celebrating Thalia's 30th birthday the Technicie is launching the 'Thalia Throwback Machine' for members to look back on the past years. 'What is this and what does it have to do with photos?' you may ask.

Well, it's a website that shows photos of you! More specifically the photos that the Paparazcie has taken at the events you visited.

To be able to provide this service we had a few requirements:
 - We needed to do face recognition
 - We needed to do it fast
 - We needed to be able to run this on a cheap AWS EC2 instance

This blog explains our approach to realise this project in production for the 500-ish members that Thalia currently has.

## A quick solution for face recognition

Even though we have a Data Science department at the university this does not mean that we really need to do face detection from scratch. People have created solutions before, so we are using one of those!

Since Python is not only the data scientist's language of choice, we also use it for our websites. The main Thalia website is built on the Django web framework, and thus this face recognition app would be as well. That meant that we also needed a Python library to do the face recognition: https://github.com/ageitgey/face_recognition.

A simple and effective solution that allows us to easily recognise the faces of all our members! 

```python
picture_of_member = face_recognition.load_image_file("member.jpg")
member_face_encoding = face_recognition.face_encodings(picture_of_member)[0]

unknown_picture = face_recognition.load_image_file("unknown.jpg")
unknown_face_encoding = face_recognition.face_encodings(unknown_picture)[0]

results = face_recognition.compare_faces([member_face_encoding], unknown_face_encoding)
```

Just five lines of code and we have a working basis for face recognition. Now we only need to scale it up.

## Reducing calculation time

The code we used above is pretty quick if you only need to do face recognition for one person on one photo. With possibly over one hundred members finding their own faces in our database of thousands of photos we need a different solution.

First, we are changing what we are searching for. Instead of having to extract the face encodings every time we need to find a member in the haystack of photos we are extracting the face encodings from all the photos. A long import process, but it will definitely be worth it in the end. All the face encodings are saved to our PostgreSQL database. With our models filled with the 128 values from the encoding we have all the information we need to find the faces of our members. At least, almost.

```python
def obtain_encodings(image_id, album_id, image_file):
    img = Image.open(image_file)
    img = img.convert("RGB")
    # We downscale the image to increase face detection speed
    img.thumbnail((500, 500))
    encodings = face_recognition.face_encodings(np.array(img))

    for encoding in encodings:
        model = FaceEncoding(image_id=image_id, album_id=album_id)
        for i in range(0, 128):
            setattr(model, f"field{i}", encoding[i])
        model.save()
```

To be able to find a face of a member we need to have a source image, something to compare to, and we need a way to compare the source image to the encodings we previously obtained.

For the source image we let the members upload pictures of themselves, since we do not want to label all of their faces ourselves. We can repeat the code we used before and connect the encoding that we obtain to the user account of the member. 

Now we need to compare the images the member uploaded to all the photos in our database. Lucky for us databases are made to quickly retrieve information based on queries. What a coincidence, isn't it? Time to write our query!

To know what kind of query we need, we need to look into the internals of the `face_recognition.compare_faces` function. The documentation gives us the answer: we need to calculate the [Euclidian distance](https://en.wikipedia.org/wiki/Euclidean_distance). 

```python
encoding = Member.objects.get_face_encoding()
distance_function = "sqrt("
for i in range(0, 128):
    distance_function += f"power(field{i} - {encoding[i]}, 2) + "
distance_function = distance_function[0:-2] + ")"

matches = FaceEncoding.objects.extra(where=[distance_function} + " < 0.49"])
```

That's it. We now know exactly what photos from our database we need to show to the member.

## Shipping it!

In practice there are several other things that we needed to do to be able to run this in production. Caching of the cryptographically signed photo urls, for example. The photos are not located on the server that we use to run this application on, so we need to get the actual images from our main website to show them to our members.

Want to see the complete code of our throwback machine? It's available on [GitHub](https://github.com/svthalia/face-detect-app)
