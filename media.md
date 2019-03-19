# References

public class MediaService extends Service implements MediaPlayer.OnBufferingUpdateListener,
                                                     MediaPlayer.OnCompletionListener,
                                                     MediaPlayer.OnInfoListener, MediaPlayer
                                                             .OnPreparedListener, MediaPlayer
                                                             .OnSeekCompleteListener {

    DataListener mCallback;
    private final String TAG = "MediaService";
    IBinder binder = new MyBinder();
    boolean allowRebind, trackEnded = false;

    private List<Track> tracks;
    MediaPlayer mPlayer;
    int totalTracks, curTrack;

    public MediaService() {
        Log.d(TAG, "Constructor called");
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.d(TAG, "onCreate");
        new TracksLoader().execute();
        mPlayer = new MediaPlayer();
        mPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);

        mPlayer.setOnBufferingUpdateListener(this);
        mPlayer.setOnCompletionListener(this);
        mPlayer.setOnInfoListener(this);
        mPlayer.setOnPreparedListener(this);
        mPlayer.setOnSeekCompleteListener(this);
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.d(TAG, "onStartCommand");
        return START_STICKY;
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        bindCount++;
        Log.d(TAG, "onBind" + bindCount);
        return binder;
    }

    @Override
    public void onRebind(Intent intent) {
        Log.d(TAG, "onRebind");
        super.onRebind(intent);
    }

    private int bindCount = 0;
    @Override
    public boolean onUnbind(Intent intent) {
        --bindCount;
        Log.d(TAG, "onUnbind" + bindCount);
        if(bindCount == 0 && !isPlaying())
            onDestroy();
        return allowRebind;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy");
        mPlayer.release();
        mPlayer = null;
        mCallback = null;
    }

    @Override
    public void onBufferingUpdate(MediaPlayer mp, int percent) {
        if (mCallback != null)
            mCallback.OnBuffered(mp, percent);
        else
            Log.d(TAG, "mCallback = null");
    }

    @Override
    public void onCompletion(MediaPlayer mp) {
        Log.d(TAG, "onCompletion");
        trackEnded = true;
        playNextTrack();
    }

    @Override
    public boolean onInfo(MediaPlayer mp, int what, int extra) {
        Log.d(TAG, "onInfo");
        return false;
    }

    @Override
    public void onPrepared(MediaPlayer mp) {
        Log.d(TAG, "onPrepared");
        mp.start();
    }

    @Override
    public void onSeekComplete(MediaPlayer mp) {
        Log.d(TAG, "onSeekComplete");
    }

    public void seekTo(int position) {
        mPlayer.seekTo(position);
    }

    public boolean isPlaying() {
        return mPlayer != null && mPlayer.isPlaying();
    }

    public void setCallback(DataListener mCallback){
        this.mCallback = mCallback;
    }
    public void play(int curTrack) {
        Log.d(TAG, "play");
        if (mPlayer.isPlaying()) {
            mPlayer.stop();
        }
        mPlayer.reset();
        try {
            mPlayer.setDataSource(getTrackUrl(curTrack));
            mPlayer.prepareAsync();
            this.curTrack = curTrack;
            mCallback.onPlay(tracks.get(curTrack));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public void playPreviousTrack() {
        Log.d(TAG, "playPreviousTrack");
        if (mPlayer.isPlaying()) {
            if (curTrack == 0) {
                play(totalTracks - 1);
            } else {
                play(curTrack - 1);
            }
        }
    }

    public void playNextTrack() {
        Log.d(TAG, "playNextTrack");
        if (mPlayer.isPlaying() || trackEnded) {
            trackEnded = false;
            if (curTrack == tracks.size())
                play(0);
            else
                play(curTrack + 1);
        }
    }



    public void pause() {
        Log.d(TAG, "pause");
        mPlayer.pause();
    }

    public void resume() {
        Log.d(TAG, "resume");
        mPlayer.start();
    }

    public void stop() {
        Log.d(TAG, "stop");
        if (mPlayer.isPlaying())
            mPlayer.stop();
        mPlayer.reset();
    }

    private String getTrackUrl(int position) {
        Log.d(TAG, "getTrackUrl");
        return (tracks.get(position).getStream_url() + "?client_id=" + ConnectionUtils.CLIENT_ID);
    }

    public void togglePlay(ImageButton bt) {
        if (mPlayer.isPlaying()) {
            mPlayer.pause();
        } else {
            mPlayer.start();
            bt.setImageResource(R.drawable.ic_pause);
        }
    }

    public class MyBinder extends Binder {

        MediaService getService() {
            Log.d(TAG, "MyBinder.getService");
            return MediaService.this;
        }
    }

    private class TracksLoader extends AsyncTask<Void, Void, Track[]> {

        @Override
        protected Track[] doInBackground(Void... params) {
            Log.d(TAG, "TracksLoader.doInBg");
            Track[] tracks = null;
            DatabaseAdapter db = new DatabaseAdapter();
            db.open(true);
            try {
                tracks = db.getTracks();
            } finally {
                db.close();
            }
            return tracks;
        }

        @Override
        protected void onPostExecute(Track[] newTracks) {
            super.onPostExecute(newTracks);
            Log.d(TAG, "TracksLoader.onPost");
            tracks = Arrays.asList(newTracks);
            totalTracks = newTracks.length;
        }
    }
}
