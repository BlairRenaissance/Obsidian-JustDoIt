
The fastest method for performing ray/AABB intersections is **the slab method**. The idea is to treat the box as the space inside of three pairs of parallel planes. The ray is clipped by each pair of parallel planes, and if any portion of the ray remains, it intersected the box.

![](https://tavianator.com/2011/slab_method.gif)

A simple implementation of this algorithm might look like this (in two dimensions for brevity):

```
bool intersection(box b, ray r) {
    double tmin = -INFINITY, tmax = INFINITY;

    if (ray.n.x != 0.0) {
        double tx1 = (b.min.x - r.x0.x)/r.n.x;
        double tx2 = (b.max.x - r.x0.x)/r.n.x;

        tmin = max(tmin, min(tx1, tx2));
        tmax = min(tmax, max(tx1, tx2));
    }

    if (ray.n.y != 0.0) {
        double ty1 = (b.min.y - r.x0.y)/r.n.y;
        double ty2 = (b.max.y - r.x0.y)/r.n.y;

        tmin = max(tmin, min(ty1, ty2));
        tmax = min(tmax, max(ty1, ty2));
    }

    return tmax >= tmin;
}
```

However, those divisions take quite a bit of time. Since when ray-tracing, the same ray is tested against many AABBs, it makes sense to pre-calculate the inverses of the direction components of the ray. If we can rely on the IEEE 754 floating-point properties, this also implicitly handles the edge case where a component of the direction is zero - the `tx1` and `tx2` values (for example) will be infinities of opposite sign if the ray is within the slabs, thus leaving `tmin` and `tmax` unchanged. If the ray is outside the slabs, `tx1` and `tx2` will be infinities with the same sign, thus making `tmin == +inf` or `tmax == -inf`, and causing the test to fail.

The final implementation would look like this:

```
bool intersection(box b, ray r) {
    double tx1 = (b.min.x - r.x0.x)*r.n_inv.x;
	double tx2 = (b.max.x - r.x0.x)*r.n_inv.x;

    double tmin = min(tx1, tx2);
    double tmax = max(tx1, tx2);

    double ty1 = (b.min.y - r.x0.y)*r.n_inv.y;
    double ty2 = (b.max.y - r.x0.y)*r.n_inv.y;

    tmin = max(tmin, min(ty1, ty2));
    tmax = min(tmax, max(ty1, ty2));

    return tmax >= tmin;
}
```

Since modern floating-point instruction sets can compute min and max without branches, this gives a ray/AABB intersection test with no branches or divisions.

三维示例如下：

```
#include <iostream>
#include <glm/glm.hpp>

bool rayAABBIntersection(const glm::vec3& rayOrigin, const glm::vec3& rayDirection, const glm::vec3& aabbMin, const glm::vec3& aabbMax) {
    glm::vec3 tmin = glm::vec3(-FLT_MAX, -FLT_MAX, -FLT_MAX);
    glm::vec3 tmax = glm::vec3(FLT_MAX, FLT_MAX, FLT_MAX);

    for (int i = 0; i < 3; ++i) {
        if (glm::abs(rayDirection[i]) < 1e-6) {
            if (rayOrigin[i] < aabbMin[i] || rayOrigin[i] > aabbMax[i]) {
                return false;
            }
        } else {
	        // 假设射线的i方向到达min平面的时间为t1
            float t1 = (aabbMin[i] - rayOrigin[i]) / rayDirection[i];
            // 假设射线的i方向到达max平面的时间为t2
            float t2 = (aabbMax[i] - rayOrigin[i]) / rayDirection[i];
            tmin[i] = std::max(tmin[i], std::min(t1, t2));
            tmax[i] = std::min(tmax[i], std::max(t1, t2));
        }
    }

	// 出范围的时间晚于进范围的时间则相交
	// 想象一下射线和一个正方形不相交的条件，一定是先从一个延伸边穿出去了，再进入的另一个延伸边，才不相交，解释图见下
    return glm::min(glm::min(tmax.x, tmax.y), tmax.z) >= glm::max(glm::max(tmin.x, tmin.y), tmin.z); 
}

int main() {
    // 示例使用
    glm::vec3 rayOrigin(0.0f, 0.0f, 0.0f);
    glm::vec3 rayDirection(1.0f, 0.0f, 0.0f);  // 示例射线沿 x 轴正方向
    glm::vec3 aabbMin(-1.0f, -1.0f, -1.0f);
    glm::vec3 aabbMax(1.0f, 1.0f, 1.0f);

    if (rayAABBIntersection(rayOrigin, rayDirection, aabbMin, aabbMax)) {
        std::cout << "射线与AABB相交" << std::endl;
    } else {
        std::cout << "射线与AABB不相交" << std::endl;
    }

    return 0;
}

```

a. 一维时，除非射线和边平行，否则一定满足tmin<tmax。
b. 二维时，当射线不想和正方形相交，只能是先传出a才进入c。即射线和一个正方形不相交的条件，一定是先从一个延伸边穿出去了，再进入的另一个延伸边，才不相交。即存在tmin>tmax。所以相交的条件是max(tmin1, tmin2)<=min(tmax1, tmax2)。
c. 三维时，当射线想和正方体相交时，但凡有一个方向的二维投影存在tmin>tmax，那就不相交了。所以相交的条件是三个方向的投影都满足max(tmin1, tmin2)<=min(tmax1, tmax2)。合并一下就是max(tmin1, tmin2, tmin3)<=min(tmax1, tmax2, tmax3)。

![[IMG_20230911_161716.jpg]]