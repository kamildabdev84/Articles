# How to save state of multiple views in activity

Very often android apps contain screens with many subscreens  
From the UX perspective it is very important to keep state of each subscreen

How to properly save state of each subscreen?  
Many people would use fragments for this purpose  
Fragments API however is hmm... how to say it politely...  
Complicated, not intuitive, error prone and simply horrible

- HORRIBLE fragmentManager
- HORRIBLE back stack management
- HORRIBLE lifecycle methods
- HORRIBLE communication with activity
- ABSOLUTELY HORRIBLE nested fragments
- and many other situations where you can shoot in your own foot  
  
  
Many people would probably disagree with me.  
But for me fragments API is just a NIGHTMARE !!!


How to live without fragments?  
Their functionality can be simply replaced by views  
which were in android SDK from the beginning  

Saving state in android views is straightforward, there are two methods:
- **onSaveInstanceState** in which you save state to Bundle
- **onRestoreInstanceState** in which you load saved state from Bundle

Everything works fine when view has its ID and was not added dynamically  
I noticed however that for freshly created views that were dynamically added to ViewGroup  
**onSaveInstanceState** and **onRestoreInstanceState** are NOT called at all  
and the state of view cannot be saved and restored...  
Views are dynamically added and removed in PagerAdapter used by ViewPager  
so we have to deal with this problem somehow  

### SOLUTION

* Save state in view  **onDetachFromWindow**

* Restore state in view **onAttachFromWindow** 

* Clear state **when activity is finishing and not changing configuration**
	
* Store state **in memory**

Actually you could store your view state wherever you want:  
- memory
- shared preferences
- database 
- file  

Personally however I would recommend to use memory  
because disk operations are slow and it results in UI delays  
that in my opinion are bad for user experience  
Saved state should be cleared when activity is finished  
so for most cases OutOfMemoryError is not a concern  

Sometimes of course there will be a need to save state in more persistent way  
because of business requirements or amount of data to store  
however it is a lot more labor-intensive and in my opinion in most cases not needed

Often there is a need to reuse views in other screens  
View A can be used in screen X and in screen Y  
There is a need to differentiate states of view A in each screeen  
To achieve this I simply set save state key in particular view
  
Enough talking, let's jump into the code  


### ViewStateData  
Used to store states of application views and activities
  
```java
public enum ViewStateData {

    INSTANCE(new HashMap<>());

    public final HashMap<String, Object> stateMap;

    ViewStateData(HashMap<String, Object> stateMap) {
        this.stateMap = stateMap;
    }

    public void saveViewState(String key, Object state) {
        if (state == null) {
            throw new IllegalArgumentException("cannot save empty state");
        }

        if (key == null) {
            throw new IllegalArgumentException("cannot save when key is NULL");
        }

        stateMap.put(key, state);
    }

    public void clearViewState(String key) {
        if (key == null) {
            throw new IllegalArgumentException("cannot clear state when key is NULL");
        }

        stateMap.remove(key);
    }

    public Object getViewState(String key) {
        if (key == null) {
            throw new IllegalArgumentException("cannot get state when key is NULL");
        }
        return stateMap.get(key);
    }
}
```


#### SaveStateView
Implemented in view to differentiate state of view in different screens

```java
public interface SavedStateView {

    void setStateKey(String stateKey);

    void saveState();

    void restoreState();
	
}
```

#### MultiViewDisplayManager 

Used to reduce boilerplate code connected with adding and removing views from ViewGroup.  
It is used to switch beetween views. Useful when you do not use ViewPager in your activity

```java
public class MultiViewDisplayManager {

    private HashMap<String, SavedStateView> savedStateViewMap = new HashMap<>();
    private ViewGroup viewContainer;

    public MultiViewDisplayManager(ViewGroup viewContainer) {
        this.viewContainer = viewContainer;
    }

    public void clear() {
        viewContainer.removeAllViews();
        viewContainer = null;
        savedStateViewMap .clear();
    }

    public void displayView(SavedStateView savedStateView, String stateKey) {
        if (!savableViewsMap.containsKey(stateKey)) {
            savedStateViewMap.clear();
            viewContainer.removeAllViews();
            savedStateView.setStateKey(stateKey);
            savedStateViewMap.put(stateKey, savedStateView);
            View view = (View) savedStateView;
            viewContainer.addView(view);
        }
    }
}

```

#### FormView 
Example view in which described saving strategy was used

```java
public class FormView extends FrameLayout implements SavedStateView {

    private static final String FORM_IN_MAIN_STATE = "FORM_IN_MAIN_STATE";
    private static final String FORM_IN_PAGER_STATE = "FORM_IN_PAGER_STATE";

    private FormViewListener formViewListener;
    private EditText name;
    private EditText surname;
    private String stateKey;

    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        saveState();
        formViewListener.onFormViewDetached();
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        restoreState();
        formViewListener.onFormViewAttached(this);
    }

    @Override
    public void saveState() {
        FormViewState state = new FormViewState(name.getText().toString(), surname.getText().toString());
        ViewStateData.INSTANCE.saveViewState(stateKey, state);
    }

    @Override
    public void restoreState() {
        FormViewStatete state = (FormViewState) ViewStateData.INSTANCE.getViewState(stateKey);
        nameInput.setText(state.name);
        surnameInput.setText(state.surname);
    }

    @Override
    public void setStateKey(String stateKey) {
        this.stateKey = key;
    }

    public void setFormViewListener(FormViewListener listener) {
        this.formViewListener = listener;
    }
}

```

#### MainActivity 
Example activity in which described saving strategy was used

```java
public class MainActivity extends AppCompatActivity implements FormViewListener {

    private static final String MAIN_ACTIVITY_STATE = "MAIN_ACTIVITY_STATE";

    @BindView(R.id.main_view_container) FrameLayout viewContainer;
    private MultiViewDisplayManager multiViewDisplayManager;
    private FormView formView;
    private ActiveScreen activeScreen;

    public enum ActiveScreen {
        FORM, OTHER, ANOTHER
    }

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_multi_view);
        ButterKnife.bind(this);
        multiViewDisplayManager = new MultiViewDisplayManager(viewContainer);
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        //
        //	HERE I SAVE ACTIVITY STATE
        //
        MainActivityState state = new MainActivityState(activeScreen);
        ViewStateData.INSTANCE.saveViewState(MAIN_ACTIVITY_STATE, state);
    }

    @Override
    protected void onRestoreInstanceState(Bundle savedInstanceState) {
        super.onRestoreInstanceState(savedInstanceState);
        MainActivityState state = (MainActivityState) ViewStateData.INSTANCE.getViewState(MAIN_ACTIVITY_STATE);
        if (state != null) {
            //
            //	HERE I RESTORE ACTIVITY STATE
            //
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        multiViewDisplayManager.clear();
        formView = null;

        if (!isChangingConfigurations()) {
            ViewStateData.INSTANCE.clearViewState(FormView.FORM_IN_MAIN_STATE);
            ViewStateData.INSTANCE.clearViewState(MAIN_ACTIVITY_STATE);
        }
    }

    @OnClick(R.id.form_button)
    protected void formClicked() {
        FormView formView = new FormView(this);
        formView.setFormViewListener(this);
        multiViewDisplayManager.displayView(firstView, FormView.FORM_IN_MAIN_STATE);
    }

    @Override
    public void onFormViewAttached(FormView formView) {
        this.formView = formView;
        this.activeScreen = ActiveScreen.FORM;
    }

    @Override
    public void onFormViewDetached() {
    }
}

```

#### SamplePagerAdapter 
Example pager adapter in which described saving strategy was used

```java
public class MultiViewsPagerAdapter extends PagerAdapter {

    private FormViewListener formViewListener;
    private SecondViewListener secondViewListener;
    private ThirdViewListener thirdViewListener;

    @Override
    public Object instantiateItem(ViewGroup container, int position) {
        switch (position) {
            case 0:
                FormView firstView = new FirstView(container.getContext());
                formView.setStateKey(FormView.FORM_IN_PAGER_STATE);
                formView.setFormViewListener(firstViewEventListener);
                container.addView(firstView);
                return firstView;
            case 1:
                SecondView secondView = new SecondView(container.getContext());
                secondView.setStateKey(SecondView.SECOND_IN_PAGER);
                secondView.setSecondViewListener(secondViewEventListener);
                container.addView(secondView);
                return secondView;
            default:
                ThirdView thirdView = new ThirdView(container.getContext());
                thirdView.setStateKey(ThirdView.THIRD_IN_PAGER);
                thirdView.setThirdViewListener(thirdViewEventListener);
                container.addView(thirdView);
                return thirdView;
        }
    }

    @Override
    public void destroyItem(ViewGroup container, int position, Object object) {
        container.removeView((View) object);
    }

    @Override
    public boolean isViewFromObject(View view, Object object) {
        return view == object;
    }

    @Override
    public int getCount() {
        return 3;
    }

    public void setFormViewListener(FormViewListener formViewListener) {
        this.formViewListener = formViewListener;
    }

    public void setSecondViewListener(SecondViewListener secondViewListener) {
        this.secondViewListener = secondViewListener;
    }

    public void setThirdViewListener(ThirdViewListener thirdViewListener) {
        this.thirdViewListener = thirdViewListener;
    }
}

```

If you would like to use MVP architecture  
you can put HashMap with saved states in model layer  
which is always good practice  
however for simplicity of this article I omitted this step  
  
  
  
That's all. Thanks for reading  
Hope you like this implementation  
  
  

Best regards  
  

