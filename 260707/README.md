# 260707

## tasks

- move the camera to be more down and out so the arm is blocking less of the view
- two camera setup (with different launchfiles, one for the one fixed camera setup, one for the two fixed camera setup)
- continue set up high level vlm as a ros node, make images work
- test the vlm

misc:

- figure out python lsp?

## sources

1. [openai vision docs](https://developers.openai.com/api/docs/guides/images-vision)
2. [publishing and subscribing images](https://wiki.ros.org/cv_bridge/Tutorials/ConvertingBetweenROSImagesAndOpenCVImagesPython)
3. [ros message filters](https://wiki.ros.org/message_filters)
4. [creating custom ros message](https://wiki.ros.org/ROS/Tutorials/CreatingMsgAndSrv#Creating_a_msg)
5. [rosbag](https://wiki.ros.org/rosbag/Commandline)
6. [gazebo models](https://app.gazebosim.org/fuel/models)

## learned

- do not name your script the same name as your package, you will have issues
- very important to check names and types of topics when your code doesn't work
- very useful: `latch=True` on the publisher to make the message saved. ideal for slow changing data

### queue_size

how many messages ROS is allowed to buffer before dropping old ones, in the event of publishing being faster than receiving

this is an argument you can set in both publisher and subscriber. they each have their own queue

we set it to one for vision because we want the latest image (lower latency)

## log

### two camera setup

actually, realized i can make it an argument passed to the launch file

we will defer this to later however

### images on vlm node

for some reason i had to use the completions api (`client.chat.completions.create`) instead of responses api (`client.responses.create`)

here's the general shape of what i did for a proof of concept base64 image vision+language inference

```python
def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode("utf-8")

client = OpenAI(
    api_key="dummy",
    base_url="http://localhost:49173/v1"
)

rospack = rospkg.RosPack()
image_path = os.path.join(rospack.get_path('vlm'), 'assets', 'leclerc.jpg')
base64_image = encode_image(image_path)

response = client.chat.completions.create(
    model="Qwen/Qwen3.5-4B",
    messages=[
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "Who is in this image?"},
                {
                    "type": "image_url",
                    "image_url": {
                        "url": f"data:image/jpeg;base64,{base64_image}"
                    },
                },
            ],
        }
    ],
)

print(response.choices[0].message.content)
```

### camera feed on vlm node

see source 2

subscribing to the image feed and converting to cv image

```python
self.image_sub = rospy.Subscriber("/io/internal_camera/head_camera/image_raw",Image,self.callback)
```

...

```python
def callback(self,data):
    try:
        cv_image = self.bridge.imgmsg_to_cv2(data, "bgr8")
    except CvBridgeError as e:
        print(e)
        return
```

we can encode into base 64 like this:

```python
success, buffer = cv2.imencode(".jpg", cv_image)

if not success:
    rospy.logerr("Failed to encode image")
    return

self.image = base64.b64encode(buffer).decode("utf-8")
```

### deciding the prompt

i went with this prompt to test the setup with for now:

```
Respond with only one of these phrases describing the quadrant of the image that the banana is in:
- top left
- top right
- bottom left
- bottom right
```

i chose it because it's simple and easy to verify visually

### observation about the large variation in inference time

i tried to run the above prompt at 0.1 hz (every 10 seconds), but it had a lot of variance in latency, e.g. from 5 seconds up to 90 seconds.

this is because the model sometimes takes little time to think, sometimes a lot of time. we decided we are fine with this for now, since it's a limitation in the model itself that we can't really control.

### separate ros node to publish prompt, and listen to it at the same time as the image (synchronized)

see source 3 and 4

- we add a node called "instruction" to the vlm package
- it publishes a custom StampedString message containing the instruction (string) and a Header (needed for synchronization) at 50hz
- the idea is that you could run this node, or you could publish your own message at that topic whenever you wanted to give custom instruction
- we sync up our instruction subscription and image subscription using message_filters ApproximateTimeSynchronizer

```python
instruction_sub = message_filters.Subscriber("/vlm/instruction", StampedString)
image_sub = message_filters.Subscriber("/io/internal_camera/head_camera/image_raw", Image)

self.sync = message_filters.ApproximateTimeSynchronizer(
    [instruction_sub, image_sub],
    queue_size=1,
    slop=0.05 # how far apart messages are allowed to be to be considered synchronized, in seconds
)

self.sync.registerCallback(self.callback)
```

### publishing the used image/instruction pair

- made publishers for each of them
- i also made their relationship explicit by putting them in `latest_pair` tuple together
- also improved error handling and error messages
- very useful: `latch=True` on the publisher to make the message saved, so stuff like rqt_image_view can see it

### resolving variance in inference time

i was able to increase frequency to 0.5 hz (every 2 seconds). the model now takes ~0.5s to 2s per prompt. i was able to do this by setting `reasoning_effort="none"`, and letting the model think using non-reasoning tokens, and using only the last line as its final output.

here is the prompt, with this change and a couple more tweaks (table relative, none as an option):

```
Find the quadrant of the table that the banana is in.

Your final answer is the last line of your response. It must be one of these phrases:
- top left
- top right
- bottom left
- bottom right
- none
```

### publishing model output

i processed the raw output to strip until the last line, and wrote a publisher with `StampedString` for the model output, at `/vlm/output`

### the setup now

we now have a pick and place robot that moves a banana down bit by bit periodically (that's what i eventually settled on with the pnp code), and a vision language model on a ros node that's able to identify the general spatial location of the banana in the frame, taking in the image feed topic as input and outputting a string topic.

based off vibes the model is somewhat accurate, except it is quite inconsistent when it's near the center since top/bottom is ambiguous.

### more object models (eg cup)

got from source 6

what I had to update after copying into models/:

- gazebo_object_tf.py to log tf
- corrected relative paths in `model.mtl`
- since they were sdl, we use `model://` paths in gazebo:
  ```
  - <uri>meshes/model.obj</uri>
  + <uri>model://plate/meshes/model.obj</uri>
  ```
- must pass in `-sdf` as argument in launchfile

### setting up an evaluation pipeline

now that we have a working model with easily changeable instructions and environment, we decided it was useful to set up an evaluation pipeline to essentially empirically test the accuracy of these models given certain prompts. this is useful because we will likely use a similar pipeline in the future with the full VLM + low level controller setup.

what needs to happen:

- establish a ground truth: with our current prompt, the ground truth can be found by getting the relative position of the banana to the table, and comparing it to the origin
- establish data collection: perhaps rosbag (see source 5) to store select logged topics to a file
- probably more stuff. basically we need to keep in mind a proper reproducible experiment setup and document it well
- evaluation metrics/dependent variables, independent variables, statistical significance?
