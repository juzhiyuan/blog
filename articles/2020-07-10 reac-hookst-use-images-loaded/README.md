HTML 的图像元素标签 `img` 存在属性 `complete`，用于标志图像是否加载完成（注：不一定加载成功），根据 [MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement/complete) 描述，若满足以下任意条件，则该值为 `true`，否则为 `false`：
- 指定了`src` 或 `srcSet` 属性；
- 未指定 `srcSet` 且 `src` 为空字符串；
- 图像资源已经被完整的获取（本地、网络），并等待渲染或 Compositing。
- The image element has previously determined that the image is fully available and ready for use.
- 图像加载失败。

从上述条件可知， `complete === true` 并不代表加载成功，需要结合其它条件：
1. 根据 `imageElement.onload` 与 `imageElement.onerror`；
2. `Boolean(imageElement.complete && imageElement.naturalWidth > 0)`；

延伸一下，存在这样的需求：对于一组图像，加载完毕后显示图像（成功 或 失败）；未完成时，显示 `Loading` 状态。本文通过构建自定义的 React Hook 来实现该需求。

先看代码：

```jsx
import {useEffect, useState, useRef} from 'react'

const useImagesLoaded = () => {
	const ref = useRef(null)
	const [loaded, setLoaded] = useState(false)

	useEffect(() => {
		if (!ref.current) {
			return
		}

		const resolveReferences = []
		const elements = ref.current.getElementsByTagName('img')
		const promises = [...elements].map(img => {
			if (!img.complete) {
				return new Promise(resolve => {
					resolveReferences.push(resolve)
					img.addEventListener('load', resolve, {once: true})
				})
			}
			return null
		})

		if (promises.length > 0) {
			Promise.all(promises).then(() => setLoaded(true))
		}

		return () => {
			elements.forEach((img, index) => {
				img.removeEventListener('load', resolveReferences[index])
			})
		}
	}, [ref])
	
	return [ref, loaded]
}

export default useImagesLoaded
```

以及使用方式：

```jsx
const Page = () => {
	const [ref, loaded] = useImagesLoaded()

	return (
		<div ref={ref}>
      {loaded ? <h1>Loaded</h1> : <h1>Loading</h1>}
		  <img src="src1" alt="img1" />
		  <img src="src2" alt="img2" />
		  <img src="src3" alt="img3" />
		</div>
	)
}
```

从代码可见，原理不复杂：构建一个 Promise 化的图像元素数组，使用 `Promise.all` 等待所有图像元素的 element 属性标记为 true 时，更新 loaded 状态。

## 参考
- [use-images-loaded](https://github.com/frzkn/use-images-loaded)
