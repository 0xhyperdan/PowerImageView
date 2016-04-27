# PowerImageView
可播放GIF图的ImageView自定义控件-郭神博客
#核心类
''package qiqi.love.you;

import android.content.Context;
import android.content.res.TypedArray;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.graphics.Canvas;
import android.graphics.Movie;
import android.os.SystemClock;
import android.util.AttributeSet;
import android.util.Log;
import android.util.TypedValue;
import android.view.View;
import android.widget.ImageView;

import java.io.InputStream;
import java.lang.reflect.Field;

/**
 * Created by iscod on 2016/4/27.
 */
public class PowerImageView extends ImageView implements View.OnClickListener {
    private static final String TAG = "PowerImageView";
    /**
     * 播放GIF动画的关键类
     */
    private Movie mMovie;
    /**
     * 开始播放按钮图片
     */
    private Bitmap mStartButton;
    /**
     * 记录动画开始时间
     */
    private long mMovieStart;
    /**
     * GIF图片的宽度
     */
    private int mImageWidth;
    /**
     * GIF图片的高度
     */
    private int mImageHeight;
    /**
     * 图片是否在播放
     */
    private boolean isPlaying;
    /**
     * 是否允许自动播放
     */
    private boolean isAutoPlay;

    public PowerImageView(Context context) {
        super(context);
        Log.d(TAG, "PowerImageView     " + 1);
    }

    public PowerImageView(Context context, AttributeSet attrs) {
        //this(context, attrs, 0);
        super(context, attrs);
        Log.d(TAG, "PowerImageView     " + 2);
        initView(context, attrs);
    }

    /**
     * 构造方法，在这里完成初始化
     *
     * @param context
     * @param attrs
     * @param defStyleAttr
     */
    public PowerImageView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        Log.d(TAG, "PowerImageView     " + 3);
        initView(context, attrs);
    }

    private void initView(Context context, AttributeSet attrs) {
        this.setLayerType(View.LAYER_TYPE_SOFTWARE, null);//使用软解码，不用硬件加速
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.PowerImageView);
        int resourcedId = getResourcesID(attrs);
        Log.d(TAG, "PowerImageView resID:" + resourcedId);
        if (resourcedId != 0) {
            //当资源ID不为0时，就去获取该资源流
            InputStream is = getResources().openRawResource(resourcedId);
            //实用movie类进行解码
            mMovie = Movie.decodeStream(is);
            if (mMovie != null) {
                //如果返回的Movie不为null，就说明是一个GIF图片，下面获取是否自动播放的属性。
                isAutoPlay = a.getBoolean(R.styleable.PowerImageView_auto_play, false);
                Bitmap bitmap = BitmapFactory.decodeStream(is);
                mImageHeight = bitmap.getHeight();
                mImageWidth = bitmap.getWidth();
                bitmap.recycle();
                if (!isAutoPlay) {
                    //当不允许自动播放的时候，得到开始播放按钮的图片，并进行注册。
                    mStartButton = BitmapFactory.decodeResource(getResources(),
                            R.mipmap.zanting);
                    setOnClickListener(this);
                }
            }
        }
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        if (mMovie != null) {
            //如果是GIF图片则重写设定PowerImageView的大小
            setMeasuredDimension(mImageWidth, mImageHeight);
        }
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (mMovie == null) {
            //如果mMovie为null说明是一张普通图片，直接调用父类的onDraw即可。
            super.onDraw(canvas);
        } else {
            //mMovie不等于null,说明是一张GIF图
            if (isAutoPlay) {
                //如果允许自动播放，就调用playMovie()方法播放GIF动画
                playMovie(canvas);
                invalidate();
            } else {
                //不允许自动播放时，判断当前图片是否正在播放
                if (isPlaying) {
                    //正在播放就继续调用playMovie（）方法，一直到动画播放结束为止。
                    if (playMovie(canvas)) {
                        isPlaying = false;
                    }
                    invalidate();
                } else {
                    //还有没有播放就只绘制GIF图片的第一帧，并绘制一个开始按钮
                    mMovie.setTime(0);
                    mMovie.draw(canvas, 0, 0);
                    int offsetW = (mImageWidth - mStartButton.getWidth()) / 2;
                    int offsetH = (mImageHeight - mStartButton.getHeight()) / 2;
                    canvas.drawBitmap(mStartButton, offsetW, offsetH, null);
                }
            }
        }
    }

    /**
     * 开始播放GIF动画，播放完成返回true，未完成返回false
     *
     * @param canvas
     * @return 播放完成返回true，未完成返回false
     */
    private boolean playMovie(Canvas canvas) {
        long now = SystemClock.uptimeMillis();
        if (mMovieStart == 0) {
            mMovieStart = now;
        }
        int duration = mMovie.duration();
        //Log.d(TAG, "playMovie: duration    " + duration);
        if (duration == 0) {
            duration = 1000;
        }
        int relTime = (int) ((now - mMovieStart) % duration);
        mMovie.setTime(relTime);
        mMovie.draw(canvas, 0, 0);
        if ((now - mMovieStart) >= duration) {
            mMovieStart = 0;
            return true;
        }
        return false;
    }

    /**
     * 通过Java反射，获取到src指定图片资源所对应的id；
     * !!!!!!!!!!!!!!!!4.4以上系统不能用
     *
     * @param a
     * @param context
     * @param attrs
     * @return 返回布局文件中指定图片资源所对应的id，没有指定任何图片资源就返回0
     */
    private int getResourcesId(TypedArray a, Context context, AttributeSet attrs) {
        try {
            Field field = TypedArray.class.getDeclaredField("mValue");
            field.setAccessible(true);
            TypedValue typedValueObject = (TypedValue) field.get(a);
            return typedValueObject.resourceId;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (a != null) {
                a.recycle();
            }
        }
        return 0;
    }

    private int getResourcesID(AttributeSet attrs) {
        for (int i = 0; i < attrs.getAttributeCount(); i++) {
            if (attrs.getAttributeName(i).equals("src")) {
                return attrs.getAttributeResourceValue(i, 0);
            }
        }
        return 0;
    }

    @Override
    public void onClick(View v) {
        if (v.getId() == getId()) {
            isPlaying = true;
            invalidate();//视图重绘，如果我们想要手动地强制让视图进行重绘，可以调用invalidate()方法来实现。
        }
    }
}''

