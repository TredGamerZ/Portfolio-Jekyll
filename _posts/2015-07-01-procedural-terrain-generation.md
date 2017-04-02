---
layout: post
title: Procedural Terrain Generation
img: terr.png
priority: 1
type: p
---

The main purpose of this project was to create a tool which can be used by designers to generate controlled procedural terrains.
We used a agent based approach to generate terrains which have configurable parameters yet generate realistic mountains.
Qt was used for making gui by which designers could easily change the parameters.

This project was done by Ashish aapan and myself in our junior year as a course project for _Computer Graphics_ course.
You can find the **source code** on my friend [github repository][repo-link]

***

Most aspects of terrains can be represented as two-dimensional matrices of real numbers. The width and height of the matrix maps to the `x` and `z` dimensions of rectangular surface. In case of terrains, the value of each cell corresponds to height of the terrain (over some baseline) at that point. This rectangular grid is known as a `height map`. We used height maps to generate our terrains. Other representation such as voxels can also provide the ability to model overhangs and caves which a height map doesnot. \\
But let stick to height map and see how can we create a terrain having some kind of mountains in middle and beaches as we reach toward the boundaries.

At core we used two classes for heightmap generation first is `CoastlineAgent` and second`MountainAgent`.

The `CoastlineAgent` job is to generate a an random island.
{% highlight c++ %}
void CoastlineAgent::doWork() {
    //srandom(time(NULL));
    int w = maxX - minX + 1;
    int h = maxY - minY + 1;
    if( w*h <= patchSize){
        int dir = (int) (random()%8);
        std::pair<int, int> point = choosePoint(dir);
        std::pair<int, int> attr = std::make_pair(random()%(width), random()%(height));
        std::pair<int, int> repul = std::make_pair(random()%(width), random()%(height));
        while(tokens > 0 && point.first != -1 && point.second != -1){


            double score = -1e9;
            int mxx=0,mxy=0;
            for(int j=0; j<8; j++){
                int xx = point.first + dirList[j][0];
                int yy = point.second + dirList[j][1];
                if(isValid(xx,yy) && !isLand(xx, yy)){
                    std::pair<int, int> pt = std::make_pair(xx,yy);
                    double sp = distance(pt,repul) - 2*distance(pt, attr) + 3*minDistFromEdge(pt);
                    if(sp > score){
                        mxx = xx;
                        mxy = yy;
                        score = sp;
                    }
                }
            }
            if(score > CoastlineAgent::thresholdValue){
                image.at<uchar>(mxy,mxx) = 255;
            }
            point = choosePoint(dir);
            tokens--;
        }
    }else{

        int add=0;
        if(tokens & 1)add= 1;

        CoastlineAgent *child1,*child2;
        if(w > h){
            child1 = new CoastlineAgent(tokens/2 ,image, width, height, minX,minX+w/2,minY, maxY);
            child2 = new CoastlineAgent(tokens/2 + add,image, width, height, minX+w/2 +1, maxX, minY, maxY);
        }else{
            child1 = new CoastlineAgent(tokens/2,image, width, height, minX, maxX, minY,minY+h/2);
            child2 = new CoastlineAgent(tokens/2 + add,image, width, height, minX, maxX, minY + h/2 + 1, maxY);
        }
        child1->doWork();
        child2->doWork();
        //TODO free child data in recursion here
        delete child1;
        delete child2;
    }
}
{% endhighlight %}

The CoastlineAgent subdivide the work for the given image until it have a small portion of image. When it hits the base case.
it randomly takes two points **Attractor** and **Repulsor**. After that it loops over until all the tokens are used(tokens are the amount of points it will mark land in the specific portion of the image). Every time a loop is executed one point is selected and a score is calculated for that point. If the score is below a threshold then its left as it is otherwise it is marked as a land.

After the Coastline Agent does its job, gaussian blur and box blur are applied on the image to make the noise dissapear and so that island appear more round in shape. We also used a threshold which can be used to change the size of the island produced at the end.

After implementing the GUI, these were the result.

![Coastline Agent]({{site.baseurl}}/images/terrain1.png)

This image is then passed two `MountainAgent`. To make mountains following algorithm is used:

1. Make random zigzag lines which will denote the vallies on the island.
2. Now take the distance transform of this image.

>**What does distance transform do?**\\
>In the image the pixels are given gray values on the basis of the distance. This distance is the minimum distance from any pixel with black color(Background color).

So the idea behind the distance transform is that it will elevate the land which are furthermost from the coast boundaries and the the valley lines, and we want exactly this. The points which are closer to the coastline boundaries or to the valley line should be at lower level as compared to the points that are further away.

After making the valley lines on the coast map. This piece of code is called.
{% highlight c++ %}
void MountainAgent::makeMountains(cv::Mat &image) {
    cv::distanceTransform(image, image, CV_DIST_L2, 3);
    cv::normalize(image, image, 0, 1., cv::NORM_MINMAX);
}
{% endhighlight %}

But still this process generate the output like this:

![one-time-mountainagent]({{site.baseurl}}/images/imgres.jpg)
{: .text-center}

To make the terrain more realistic, we used the MountainAgent again and again, each time adding the different output images produced to the final image but with decreasing weights. The idea is to first have a low frequency details of the height map and adding high frequency details with lower amplitude in each successive iterations.

This produced the result like this:
![mountain-agent]({{site.baseurl}}/images/terrain2.png)

After this perlin noise can be added on the image as the final high level detail.(Though in these images perlin noise are not added)

This completed our height map generation.


Following images show the terrain generated from the above heightmap.
![image1]({{site.baseurl}}/images/terrain3.png)

![image2]({{site.baseurl}}/images/terrain4.png)

[repo-link]: https://github.com/Ashish424/ComputerGraphicsProject
